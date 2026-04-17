# mendix-container-resizing
Research on mendix container resizing and maintanatance page enablement

This document serves as a comprehensive technical summary and operational guide for managing the maintenance page during the upcoming container resize activity.

---

# Minutes of Technical Consultation: Mendix Maintenance Strategy
**Project:** Infrastructure Scaling (Container Resize)
**Key Objective:** Ensure zero-interruption of branding during container destruction and recreation.

---

## 1. Core Technical Concepts
The team must distinguish between the **Application Runtime** and the **Container Infrastructure**.

* **The Container Resize:** This is an infrastructure-level event managed by Mendix Support. The existing container is destroyed and a new one with larger resources is provisioned.
* **The Maintenance Page:** This is a **Network-Layer** feature. It is hosted on the Mendix Load Balancer, not within the application code.
* **Decoupling:** Because the Load Balancer serves the page, it remains visible to users even when the container is completely offline or being deleted.

---

## 2. Developer Guide: Creating the Custom Page
The page must be built with a **"Zero-Dependency"** architecture. Since the application will be down, the page cannot pull any assets from the Mendix theme or resources.

### **Mandatory Constraints:**
* **Single File:** Must be a single `.html` file.
* **Inline CSS:** All styles must be placed within `<style>` tags in the `<head>`.
* **Image Assets:** Images must not use relative paths (e.g., `src="img/logo.png"`). They must be:
    * **Base64 Encoded:** Embedded directly in the HTML.
    * **CDN Hosted:** Pointing to an external, high-availability URL (e.g., `https://cdn.company.com/logo.png`).
* **No App-Scripts:** Do not reference any Mendix-generated Javascript files.

---

## 3. DevOps Guide: Implementation & Operations
The Maintenance Mode toggle is an **instantaneous network switch**. It does **not** require an application restart to take effect.

### **Execution Sequence:**
1.  **Upload:** Navigate to **Cloud Portal > Environments > [Env] > Details** and upload the `.html` file.
2.  **Enable (Pre-Activity):** 10 minutes before the resize window, toggle **Maintenance Mode to ON**.
3.  **The Resize:** Notify Mendix Support to proceed. They will swap the containers while the Load Balancer continues to show your custom page.
4.  **Monitor:** Watch the Environment Logs. Wait for the specific log entry: `Mendix Runtime is started`.
5.  **Disable (Post-Activity):** Once the app is confirmed healthy, toggle **Maintenance Mode to OFF**. Traffic will immediately resume to the app.

---

## 4. QA Guide: Validation Protocol
To ensure the page is "bulletproof" before the Production window, QA must perform the following "Kill Test" in a lower environment (UAT):

1.  **Visual Audit:** Enable Maintenance Mode and verify branding, fonts, and images across mobile and desktop.
2.  **Infrastructure Simulation:**
    * While Maintenance Mode is **ON**, manually **STOP** the environment in the portal.
    * **Success Criteria:** The custom page must remain visible and fully functional (no broken images) even though the application status is "Stopped."
3.  **Cache Verification:** Open the site in an Incognito window to ensure the Load Balancer is serving the newest version of the HTML file.

---

## 5. Summary Table for Stakeholders

| Action | Impact on Runtime | Requires Restart? | User Sees |
| :--- | :--- | :--- | :--- |
| **Toggle Mode ON** | None (Traffic Diverted) | No | Custom Maintenance Page |
| **Container Resize** | Container Destroyed | Yes (Infrastructure) | Custom Maintenance Page |
| **Toggle Mode OFF** | Traffic Restored | No | Live Application |

---

## 6. Post-Resize Smoke Testing (Verification)
Once the Mendix Support team completes the resize and the logs show `Mendix Runtime is started`, you must verify the application's health before allowing public traffic back in.

### **How to Access the App While Maintenance Mode is ON**
In the Mendix Public Cloud, the Maintenance Mode toggle is an "all-or-nothing" switch for the primary URL. To perform a smoke test, you have two primary options:

#### **Option A: The Environment-Specific URL (Recommended)**
Mendix often provides a secondary, technical URL for environments that bypasses certain custom domain configurations.
* **Check the Portal:** Go to **Environments > [Env] > Details**. 
* **The "Default" URL:** Check if the default Mendix cloud URL (e.g., `myapp-test.mendixcloud.com`) is accessible even if your custom vanity domain (`apps.company.com`) is showing the maintenance page. 
* *Note:* In some cloud configurations, the toggle applies to all paths. If this is blocked, move to Option B.

#### **Option B: Temporary IP/Header Whitelisting (If Supported)**
If your organization uses a dedicated Mendix Cloud Resource or a WAF (Web Application Firewall) in front of Mendix:
* Your Network/DevOps team can add a **Header Bypass**. The Load Balancer can be configured to say: *"Show Maintenance Page to everyone UNLESS the request contains the header `X-Smoke-Test: True`."*
* Using a browser extension like **ModHeader**, QA can then inject that header to "see through" the maintenance page.

#### **Option C: The "Internal" API Check**
If you cannot access the UI, you can verify the runtime health via the **Mendix Runtime API** or health check endpoints:
* Access `https://<your-app-url>/rest/health` (if you have implemented a health check microflow).
* If the Load Balancer is strictly blocking all paths, you may need to briefly toggle Maintenance Mode **OFF** during a low-traffic window (seconds), hit the URL to confirm the login screen appears, and toggle it back **ON**.

---

## 7. Validating the "Strict" Load Balancer Setup
To confirm your application is strictly behind the Load Balancer and that the Maintenance Page is behaving correctly:

### **The "Direct Hit" Test**
1.  **Identify the IP:** Use a `ping` or `nslookup` command on your application URL to find the current IP address assigned to the Load Balancer.
2.  **Trace the Route:** Use `tracert` (Windows) or `traceroute` (Mac/Linux).
    * **Verification:** You should see the traffic hitting a Mendix-owned gateway (e.g., `edge.mendix.com` or an AWS ELB endpoint) before it reaches the app.
3.  **The Header Verification:**
    * Open Browser Developer Tools (F12) > **Network Tab**.
    * Click on the main URL request.
    * Look at the **Response Headers**.
    * You should see `Server: Cloudflare`, `Server: AmazonS3`, or `X-Served-By: Mendix-LB`. 
    * If Maintenance Mode is **ON**, the `Content-Type` will usually be `text/html`, and the `Server` header will confirm it is being served by the edge, not the Mendix Runtime.

---

## 8. Updated Smoke Test Workflow for QA
1.  **Support Notification:** Support confirms resize is done.
2.  **Runtime Check:** DevOps confirms logs show `Mendix Runtime is started`.
3.  **Bypass Access:** QA accesses the app via the technical URL or Header Bypass.
4.  **Smoke Test:** * Can you reach the Login Page?
    * Does a simple "Read" operation from the Database work? (Confirms DB connection is restored).
    * Is the File Storage accessible?
5.  **Final Go-Live:** Once QA gives the "Green Light," DevOps toggles **Maintenance Mode to OFF**.

---

### **Summary for Product Owner**
> **"We are essentially creating a 'back door' for the QA team to test the new, larger engine while the 'Closed' sign is still hanging on the front door. We only open the front door once we know the engine is running smoothly."**

> **Architect's Final Note:** Ensure all Scheduled Events or long-running background processes are monitored or paused prior to the resize, as the container destruction will terminate any active Java threads immediately.
