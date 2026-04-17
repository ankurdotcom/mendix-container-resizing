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

> **Architect's Final Note:** Ensure all Scheduled Events or long-running background processes are monitored or paused prior to the resize, as the container destruction will terminate any active Java threads immediately.
