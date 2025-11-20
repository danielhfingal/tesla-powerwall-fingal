# fingal/exporter.py — tesla-powerwall-fingal v0.4.20
# The most soul-purifying Powerwall exporter ever built.

from __future__ import annotations

import asyncio
import random
import time
from datetime import datetime, UTC
from typing import Any, Dict

import orjson
from fastapi import FastAPI, HTTPException
from opentelemetry import metrics, trace
from opentelemetry.exporter.prometheus import PrometheusMetricsExporter
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.resources import Resource
from opentelemetry.sdk.trace import TracerProvider
from tenacity import retry, stop_after_attempt, wait_exponential

from pypowerwall import Powerwall


app = FastAPI(title="tesla-powerwall-fingal")

# OpenTelemetry setup
resource = Resource.create({"service.name": "tesla-powerwall-fingal"})
metrics.set_meter_provider(MeterProvider(resource=resource))
trace.set_tracer_provider(TracerProvider(resource=resource))

meter = metrics.get_meter("tesla-powerwall-fingal")
tracer = trace.get_tracer("tesla-powerwall-fingal")

exporter = PrometheusMetricsExporter()
metrics.get_meter_provider().start_pipeline(exporter.reader)
app.mount("/metrics", exporter.create_app())


# === PERFECT METRICS ===
SITE_HEARTBEAT = meter.create_counter(
    "fingal_site_heartbeat_total",
    description="Per-poll heartbeat — increase()[5m] > 0 = site alive",
)

FINGAL_UP = meter.create_gauge(
    "fingal_up",
    description="1 = exporter healthy and polling",
    unit="1",
)

FINGAL_BUILD_INFO = meter.create_up_down_counter(
    "fingal_build_info",
    description="Constant 1 with version/commit labels",
)
FINGAL_BUILD_INFO.add(1, {"version": "0.4.20", "commit": "final"})

STREAM_POLLS = meter.create_counter(
    "tesla_stream_polls_total",
    description="Poll outcomes: delta / no_change / error",
)

TESLA_LATENCY = meter.create_histogram(
    "tesla_api_latency_seconds",
    description="API call duration",
    unit="s",
)


class FingalPowerwall:
    def __init__(
        self,
        host: str | None = None,
        password: str | None = None,
        email: str = "",
        gw_pwd: str | None = None,
        fleet_api: bool = False,
        site_id: str | None = None,
    ):
        self.pw = Powerwall(
            host=host,
            password=password or "",
            email=email or "",
            gw_pwd=gw_pwd,
            cloudmode=bool(fleet_api or not host),
            siteid=site_id,
        )
        self.mode = "local" if host else ("fleet" if fleet_api else "cloud")
        self.site_id = site_id or getattr(self.pw, "siteid", "local") or "local"

        self.last_successful_poll = time.monotonic()
        self._last_state_bytes: bytes | None = None

        FINGAL_UP.set(1, {"site_id": self.site_id})

    @retry(stop=stop_after_attempt(10), wait=wait_exponential(multiplier=2, max=120))
    async def _poll_state(self) -> Dict[str, Any]:
        start = time.monotonic()
        with tracer.start_as_current_span("tesla.poll", attributes={"site_id": self.site_id, "mode": self.mode}):
            try:
                state = await asyncio.to_thread(self.pw.soestatus)
                vitals = await asyncio.to_thread(self.pw.vitals)
                state["vitals"] = vitals
                return state
            finally:
                TESLA_LATENCY.record(time.monotonic() - start, {"site_id": self.site_id})

    async def monitor_stream(self, interval: float = 8.0):
        while True:
            try:
                state = await self._poll_state()

                # Deterministic diff — no false deltas ever
                state_bytes = orjson.dumps(state, option=orjson.OPT_SORT_KEYS)
                changed = state_bytes != self._last_state_bytes
                self._last_state_bytes = state_bytes

                outcome = "delta" if changed else "no_change"
                STREAM_POLLS.add(1, {"site_id": self.site_id, "outcome": outcome})

                self.last_successful_poll = time.monotonic()
                FINGAL_UP.set(1, {"site_id": self.site_id})

            except Exception:
                STREAM_POLLS.add(1, {"site_id": self.site_id, "outcome": "error"})
                FINGAL_UP.set(0, {"site_id": self.site_id})

            SITE_HEARTBEAT.add(1, {"site_id": self.site_id})

            yield {
                **state,
                "site_id": self.site_id,
                "mode": self.mode,
                "ts": datetime.now(UTC).isoformat(),
            }

            await asyncio.sleep(interval * (1 + random.uniform(-0.2, 0.3)))


# === Global instance (replace with your real config) ===
instance = FingalPowerwall(
    host="192.168.1.42",        # ← your Gateway IP
    password="your_password",   # ← Tesla app password
    # fleet_api=True,           # ← uncomment for Fleet fallback
)


@app.get("/healthz")
async def healthz():
    if time.monotonic() - instance.last_successful_poll > 90:
        FINGAL_UP.set(0, {"site_id": instance.site_id})
        raise HTTPException(status_code=503, detail="stale poll")
    FINGAL_UP.set(1, {"site_id": instance.site_id})
    return {"status": "ok", "site_id": instance.site_id, "mode": instance.mode}


if __name__ == "__main__":
    import uvicorn
    print("tesla-powerwall-fingal v0.4.20 — carpet fringes aligned")
    uvicorn.run(app, host="0.0.0.0", port=8000)
