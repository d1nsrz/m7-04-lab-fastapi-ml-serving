# API Design Decisions

## 1. Versioning

Path-based versioning (`/v1/`) was chosen because the URL is visible to every
layer of the infrastructure stack — load balancers, API gateways, CDNs, and log
pipelines can route or deprecate on the path prefix without parsing custom
headers. Header-based versioning (e.g., `Accept: application/vnd.vismod+json;version=1`)
adds content-negotiation complexity, is invisible in browser address bars and
curl output, and receives inconsistent support from documentation and
code-generation tooling.

## 2. Batch ordering and partial failures

When `predict-batch` is called with 32 images and one is corrupt, the API
returns HTTP 200 with all 32 entries present in the `results` map — the corrupt
image receives `"status": "error"` (with an `InferenceError` describing the
failure), while the remaining 31 entries resolve normally. This partial-success
model was chosen over fail-fast because a corrupt image is an expected
data-quality issue, not a systemic error, and aborting the entire batch would
force callers to re-submit 31 valid images at additional latency and cost. The
response map is keyed by the caller-supplied `id` independent of input order, so
callers can atomically inspect `status` per entry without any position-to-id
bookkeeping.

## 3. Async lifecycle

Jobs progress through four monotonic states: `queued` (accepted, awaiting a
worker) → `processing` (inference running) → `completed` (labels populated) or
`failed` (error detail populated). Results for both terminal states are retained
for **24 hours**; after expiry, `GET /v1/predictions/{job_id}` returns `404`
with `code: "EXPIRED"` to distinguish an expired job from an ID that was never
issued. The 24-hour window accommodates downstream consumers operating across
time zones and recovering from transient polling failures, while keeping storage
costs bounded for a high-volume, multi-tenant service.
