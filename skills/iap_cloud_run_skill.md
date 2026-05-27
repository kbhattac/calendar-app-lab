# Skill: Setting up Identity-Aware Proxy (IAP) for Cloud Run

This skill documents the complete, end-to-end process for correctly setting up Identity-Aware Proxy (IAP) in front of a Cloud Run service using a Global External HTTP(S) Load Balancer. It includes all necessary workarounds for common pitfalls (like Error 52, 502s, and 401s) to ensure a successful deployment on the first attempt.

## 1. Prerequisites & OAuth Setup
1. **OAuth Brand:** An OAuth Consent Screen must be configured for the project (Internal or External).
2. **OAuth Client:** Create an OAuth Web Client ID and Secret.
   - The Authorized Redirect URI must be: `https://iap.googleapis.com/v1/oauth/clientIds/[CLIENT_ID]:handleRedirect`

## 2. Cloud Run Deployment & Security
When "Domain Restricted Sharing" is enforced, you cannot use `allUsers` to make the service public. IAP needs to authenticate users and then securely invoke Cloud Run.

1. **Deploy Cloud Run:**
   Deploy the service, restricting ingress to the Load Balancer:
   ```bash
   gcloud run deploy [SERVICE_NAME] \
     --ingress=internal-and-cloud-load-balancing \
     --region=[REGION]
   ```
   *(Note: You do not need `--no-invoker-iam-check` if you properly configure the IAP service account in step 4).*

## 3. Load Balancer Configuration
Create a Global External Load Balancer with a Serverless NEG backend.

1. **Serverless NEG:**
   ```bash
   gcloud compute network-endpoint-groups create [NEG_NAME] \
     --region=[REGION] \
     --network-endpoint-type=serverless \
     --cloud-run-service=[SERVICE_NAME]
   ```
2. **Backend Service (with IAP Enabled):**
   ```bash
   gcloud compute backend-services create [BACKEND_NAME] --global --load-balancing-scheme=EXTERNAL_MANAGED
   gcloud compute backend-services add-backend [BACKEND_NAME] --global --network-endpoint-group=[NEG_NAME] --network-endpoint-group-region=[REGION]
   gcloud compute backend-services update [BACKEND_NAME] --global \
     --iap=enabled,oauth2-client-id=[CLIENT_ID],oauth2-client-secret=[CLIENT_SECRET]
   ```
3. **URL Map & Target Proxy:**
   ```bash
   gcloud compute url-maps create [URL_MAP_NAME] --default-service=[BACKEND_NAME]
   gcloud compute target-https-proxies create [PROXY_NAME] --url-map=[URL_MAP_NAME] --ssl-certificates=[CERT_NAME]
   gcloud compute forwarding-rules create [FORWARDING_RULE_NAME] --load-balancing-scheme=EXTERNAL_MANAGED --network-tier=PREMIUM --global --target-https-proxy=[PROXY_NAME] --ports=443
   ```

## 4. IAP Service Account Provisioning & Permissions (Fix for 502 / 401 Errors)
When a user authenticates, IAP uses a Google-managed service account to invoke Cloud Run. If this account is not provisioned or lacks permissions, you will receive a 502 Bad Gateway or 401 Unauthorized.

1. **Provision the IAP Service Account:**
   If `gcloud beta services identity create` fails, use the REST API directly:
   ```bash
   curl -X POST \
     -H "Authorization: Bearer $(gcloud auth print-access-token)" \
     -H "Content-Type: application/json" \
     https://serviceusage.googleapis.com/v1/projects/[PROJECT_ID]/services/iap.googleapis.com:generateServiceIdentity
   ```
2. **Grant Cloud Run Invoker Role:**
   Find your Project Number, then grant the `run.invoker` role to the newly created IAP service account (`service-[PROJECT_NUMBER]@gcp-sa-iap.iam.gserviceaccount.com`):
   ```bash
   gcloud run services add-iam-policy-binding [SERVICE_NAME] \
     --member='serviceAccount:service-[PROJECT_NUMBER]@gcp-sa-iap.iam.gserviceaccount.com' \
     --role='roles/run.invoker' \
     --region=[REGION]
   ```
3. **Handle Global IAM Propagation Delays (Fix for 403 Forbidden with Empty Authorization Header):**
   Even after successfully applying the `run.invoker` binding, initial requests (such as a POST request to a form submit endpoint) might fail with a **403 Forbidden** error and a log message of `"Empty Authorization header value"` or `"The request was not authenticated"`.
   - **Cause:** Google Cloud IAM policy changes take **7–10 minutes** to propagate globally to all Google Front End (GFE) edge proxies. During this brief period, some GFE edge servers will still operate on the cached, outdated policy.
   - **Resolution:** Simply wait 10 minutes for global propagation. Additionally, clear your browser session cookies (using `https://[DOMAIN]/_gcp_iap/clear_login_cookie`) or use an **Incognito Window** to prevent browser-level session caching.


## 5. SSL Certificates & FQDNs (Fix for Error Code 52)
IAP **requires** a Fully Qualified Domain Name (FQDN) in the `Host` header. Raw IP addresses will be rejected with **Error Code 52: Hostname/SSL certificate mismatch**.

1. **Use a valid domain:** If you don't have a domain, use a wildcard DNS service like `nip.io` (e.g., `[IP-WITH-HYPHENS].nip.io`).
2. **Include a Subject Alternative Name (SAN):** The SSL certificate MUST have the domain in the SAN extension. A Common Name (CN) alone is insufficient and will still cause Error 52.
   ```bash
   openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
     -keyout cert.key -out cert.crt \
     -subj "/CN=[DOMAIN]" \
     -addext "subjectAltName = DNS:[DOMAIN]"
   
   gcloud compute ssl-certificates create [CERT_NAME] --certificate=cert.crt --private-key=cert.key --global
   ```

## 6. IAP Allowed Domains (Fix for Post-Auth Error 52)
Even with a valid SAN certificate, IAP restricts redirects for security. You must explicitly add your domain to the IAP `allowed_domains` list, otherwise IAP will reject the Google Sign-In redirect token with Error 52 or 401.

1. **Create an `iap-settings.yaml` file:**
   ```yaml
   accessSettings:
     allowedDomainsSettings:
       enable: true
       domains:
        - "[DOMAIN]" # e.g. [IP-WITH-HYPHENS].nip.io
   ```
2. **Apply the settings:**
   ```bash
   gcloud iap settings set iap-settings.yaml --project=[PROJECT_ID] --resource-type=compute --service=[BACKEND_NAME]
   ```

## 7. IAP User Authentication & Access (Fix for "You don't have access" Error)
Even if a user has basic high-level roles like **Project Owner** or **Project Editor**, they will receive a **"You don't have access"** screen when visiting the application unless they are explicitly granted the IAP-secured Web App User role on the resource.

1. **Grant IAP-secured Web App User Role:**
   Explicitly grant the `roles/iap.httpsResourceAccessor` role to authorized users. If resource-level policy updates on individual backend services fail due to permissions, you can leverage project-level IAM Admin permissions to grant this role at the project level:
   ```bash
   gcloud projects add-iam-policy-binding [PROJECT_ID] \
     --member='user:[USER_EMAIL]' \
     --role='roles/iap.httpsResourceAccessor'
   ```
2. **Handle Cached "Access Denied" Sessions:**
   Because browser cookies cache access decisions, a user might get a persistent "You don't have access" page even after the IAM role is granted.
   - **Force Clear Login Cookie:** Instruct the user to visit `https://[DOMAIN]/_gcp_iap/clear_login_cookie` to reset their login session.
   - **Incognito Mode:** Alternatively, test in a fresh Incognito window to force a clean authentication flow.

## Summary Checklist for Next Time:
- [ ] OAuth Client configured with IAP Redirect URI.
- [ ] Cloud Run deployed with internal/LB Ingress.
- [ ] IAP Service Agent provisioned (via curl if needed) and granted `roles/run.invoker` on Cloud Run.
- [ ] SAN-enabled SSL Certificate generated for a proper FQDN (like `[IP-WITH-HYPHENS].nip.io`).
- [ ] Global Load Balancer configured with IAP enabled on the Backend Service.
- [ ] FQDN explicitly added to the IAP Backend Service `allowed_domains` settings.
- [ ] Authorized users (including Owners/Editors) explicitly granted `roles/iap.httpsResourceAccessor` at the project or backend service level.
