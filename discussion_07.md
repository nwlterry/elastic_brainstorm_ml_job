Deploying Elastic Machine Learning in an **on-premises, air-gapped environment running v8.14.3** introduces specific operational constraints. You cannot download updates on the fly, you cannot connect to external API systems directly for ticketing, and cluster resource limitations on-prem require strict memory capping (`model_memory_limit`) to ensure you don't destabilize production nodes.

Here is your updated, air-gapped-compliant, step-by-step POC implementation plan and checklist for Elastic 8.14.3.

---

## Pre-Requisites Checklist for Air-Gapped v8.14.3

Before creating any jobs, ensure the cluster is configured to support ML locally:

* [ ] **License Check:** Verify an Enterprise or Platinum license is active (required for multi-metric/advanced anomaly detection).
* [ ] **Node Allocation:** Ensure at least one node has `node.roles: [ ml ]` enabled in `elasticsearch.yml`.
* [ ] **Kibana Configuration:** Confirm `xpack.ml.enabled: true` is set in `kibana.yml`.
* [ ] **Local Maps (Optional):** Since you are air-gapped, if your team uses geo-location data, ensure local GeoIP databases or local tile services are mapped, as external Elastic Maps calls will fail.

---

## Topic 1: Spotting "Silent Failures" (The Hidden App Glitches)

### Step-by-Step Implementation Guide (8.14.3)

1. Navigate to **Kibana > Analytics > Discover** to isolate your log format. In 8.14.3, you want to use Data Views instead of Index Patterns.
2. Go to **Kibana > Machine Learning > Anomaly Detection**. Click **Create job**.
3. Select your application Data View. In the job wizard, choose **Advanced job** configuration.
4. **Job ID:** `app_silent_failure_poc`
5. **Analysis Configuration:**
* **Bucket span:** `15m`
* **Model memory limit:** Set to `20mb` (Highly recommended for POCs in constrained on-prem clusters to prevent high RAM consumption).


6. **Detectors Block:** Click **Add detector**:
* **Function:** `low_count`
* Leave actual fields empty to count log volumes natively.


7. **Datafeed Configuration:** Expand the Datafeed block, click **Edit JSON**, and paste your explicit success query. This query filters out errors, forcing the engine to model solely your baseline of successful transactions:

```json
{
  "query": {
    "bool": {
      "must": [
        { "term": { "url.path": "/api/checkout/success" } },
        { "term": { "http.response.status_code": "200" } }
      ]
    }
  }
}

```

8. Click **Next**, review the data validation checklist provided natively by Kibana 8.14, and click **Create Job**. Choose to process the past 3 weeks of historical logs to train the model immediately.

### Air-Gapped POC Checklist

* [ ] **Data View Verification:** Confirmed that the target fields (`url.path` and `http.response.status_code`) are explicitly mapped as `keyword` or structural types in the Elastic 8.14 Data View.
* [ ] **Model Memory Cap:** Confirmed `model_memory_limit` is explicitly defined (do not leave it open on shared infrastructure).
* [ ] **Baseline Generation:** Verified the shaded model bounds tracking normal diurnal cycles via Kibana's Single Metric Viewer.

---

## Topic 2: Smart Ticket Routing (Knowing Who to Blame)

### Step-by-Step Implementation Guide (8.14.3)

1. Go to **Kibana > Machine Learning > Anomaly Detection > Create job**. Select your APM trace data or application log Data View.
2. Choose **Advanced job** configuration.
3. **Job ID:** `app_smart_routing_poc`
4. **Analysis Configuration:**
* **Bucket span:** `15m`
* **Model memory limit:** `64mb` (Slightly larger to hold partitioning states).


5. **Detectors Block:** Click **Add detector**:
* **Function:** `high_mean`
* **Field:** `transaction.duration.us` (or your log's duration field).
* **Partition field:** `service.name` (Splits the baseline tracking distinctly per microservice).


6. **Influencers configuration:** Add `service.name`, `db.system`, and `error.exception.type` to the influencers array. This populates Elastic’s local *Anomalous Inverted Index* engine to calculate statistical correlations during an incident.
7. Click **Create Job** and run it historically.

### Air-Gapped POC Checklist

* [ ] **Partition Cardinality:** Confirmed the unique count of `service.name` values is low (less than 100 distinct services) to avoid memory overruns during training.
* [ ] **Influencer Extraction:** Verified that fields like `db.system` or `error.exception.type` exist within the same documents as your latency performance metrics.
* [ ] **Correlation Test:** Navigated to the Anomaly Explorer after a performance drop and verified the local correlations panel populates without requiring external internet calls.

---

## Topic 3: Detecting "User Trap Loops" (UI Friction)

### Step-by-Step Implementation Guide (8.14.3)

1. Go to **Kibana > Machine Learning > Anomaly Detection > Create job**. Select your user log data.
2. Select **Advanced job** configuration.
3. **Job ID:** `user_trap_loops_poc`
4. **Analysis Configuration:**
* **Bucket span:** `5m` (Critical to use a small bucket size here to detect quick, repeated actions within a short execution window).
* **Model memory limit:** `40mb`


5. **Detectors Block:** Click **Add detector**:
* **Function:** `high_count`
* **By field:** `user.session.id` (Tracks session concurrency).
* **Partition field:** `url.path` (Isolates behavior on a specific screen).


6. **Influencers:** Add `url.path`.
7. Start the datafeed. Because user navigation behavior can change heavily throughout the week, let this run for at least 3-5 days to settle the noise baseline.

### Air-Gapped POC Checklist

* [ ] **Session Sanitization:** Verified that `user.session.id` is populated cleanly and does not bundle distinct users together due to local reverse-proxy configuration problems.
* [ ] **Internal Traffic Strip:** Configured the datafeed query filtering to explicitly strip out internal system health check utilities or test monitoring runners that intentionally repeat loops.

---

## Topic 4: Keeping Dashboards Simple (The "Traffic Light" Approach)

### Step-by-Step Implementation Guide (8.14.3)

1. Go to **Kibana > Analytics > Dashboard**. Click **Create dashboard**.
2. Click **Create visualization** or **Add panel**. Select the **Machine Learning Anomaly Timeline / Swimlane** panel type (native to 8.x dashboarding options).
3. Select `app_smart_routing_poc` as the data source. Partition rows by `service.name`.

4. **Air-Gapped Escalation Routing Integration:** Since you are in an air-gapped system, you cannot use webhooks out to public APIs. Instead, configure a **Custom URL** that targets an on-prem internal ticketing webpage or a pre-formatted Kibana filter view:
* Go back to the **Machine Learning Job Management** interface.
* Edit `app_smart_routing_poc` and navigate to the **Custom URLs** tab.
* Click **Add Custom URL**.
* Set **Label** to: `Generate Triage View`.
* Set the URL pattern to link directly to a pre-filtered Kibana Discover page on your on-premises network instance, using runtime token substitutions:
```text
http://your-on-prem-kibana:5601/app/discover#/?_a=(query:(language:kuery,query:'service.name:"$service.name$"'))

```





```
5. Save the panel layout. Non-technical application support staff can now open this single dashboard, view color blocks, and click a red element to see an automated view showing exactly what went wrong.

### Air-Gapped POC Checklist
* [ ] **Technical Data Hidden:** Verified that all complex line charts, mathematical bounds, and system configuration metrics are hidden from this operational team view.
* [ ] **On-Prem Link Test:** Confirmed that the Custom URL tokens (`$service.name$`) render properly and link cleanly to your internal on-prem systems without hitting external internet redirection pages.
* [ ] **Access Mapping:** Verified that the Application Support user accounts are mapped to an Elastic role that lets them view this specific Dashboard without exposing the structural ML configuration pages.

```
