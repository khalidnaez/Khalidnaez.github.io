# Auto pilot uploader integration handoff

Status: website and developer configuration work in progress; no verified local integration.

## Expected local setup
In the PRIVATE automation project (not this public website repository), confirm .env is ignored, then create .env containing TIKTOK_CLIENT_KEY and TIKTOK_CLIENT_SECRET from the same active Sandbox. Restart the dashboard and open http://localhost:8765 on that Windows machine. Never commit tokens or secrets. Do not use GitHub Pages as the OAuth callback or token server.

Desktop callback: http://localhost:8765/tiktok/callback
Scopes: user.info.basic,video.upload,video.list
Desktop PKCE: cryptographically random per-request verifier (43–128 allowed characters), S256 challenge using SHA256 HEX per TikTok Desktop documentation. Validate a short-lived, single-use OAuth state. Keep verifier bound to that state. Exchange code on the local server; never log code, tokens, secret or upload URL.

## Endpoints
- Authorization: https://www.tiktok.com/v2/auth/authorize/
- Exchange/refresh: POST https://open.tiktokapis.com/v2/oauth/token/ as application/x-www-form-urlencoded. Exchange uses client_key, client_secret, code, grant_type=authorization_code, redirect_uri, code_verifier. Refresh uses grant_type=refresh_token and returned refresh token. Persist rotated refresh tokens atomically and use returned expiries.
- Draft init: POST https://open.tiktokapis.com/v2/post/publish/inbox/video/init/ using FILE_UPLOAD source_info. Validate size and chunking against current media-transfer documentation.
- Binary transfer: PUT to exact returned upload_url with matching Content-Type, Content-Length, Content-Range. Do not leak Bearer token to arbitrary upload hosts.
- Status: POST https://open.tiktokapis.com/v2/post/publish/status/fetch/ with publish_id. SEND_TO_USER_INBOX means delivered, not published. PUBLISH_COMPLETE indicates completion; post IDs may be unavailable for nonpublic/unmoderated posts.
- Metrics: POST https://open.tiktokapis.com/v2/video/list/?fields=id,title,create_time,view_count,like_count,comment_count,share_count ; paginate. POST /v2/video/query/ with filters.video_ids (up to 20) to refresh confirmed IDs.

## Required checks in inaccessible Windows code
- Bind dashboard to loopback; protect approval endpoints against CSRF.
- No automatic queued uploads triggered merely by connecting OAuth. Explicitly confirm each upload and make duplicate delivery prevention persistent.
- Confirm actual granted scopes; handle refusal, stale state, replayed callback, expired verifier, refresh rotation and revocation.
- Check TikTok error.code even for HTTP 200; distinguish auth/scope/user restrictions from retryable failures. Redact log_id alongside any user data shared externally.
- Backoff with jitter for 429/5xx. Do not blindly reinitialize on timeout; reconcile saved publish_id first.
- Upload init limit: 6/minute per user token. Status: 30/minute per user token. Public user/video APIs: documented default 600/minute each. At most 5 pending shares per 24h; pause on pending-share restriction rather than looping.
- Never infer successful publishing from a successful PUT. Confirm inbox notification and status on approved test content.
- Link an episode using confirmed public post ID; do not rely only on title/time heuristics. Draft analytics are not available through video.list.
- Implement approved retention: uploaded local video copies 30 days; metrics 12 months; logs 30 days; delete local tokens on disconnect. The public policy explicitly states automatic enforcement remains unverified.
- Do not train/fine-tune AI on TikTok data. Only send selected counts for planning if developer terms permit; never send tokens/profile data. Review Anthropic/provider settings before enabling.

## Test matrix (NOT RUN)
OAuth grant/refusal and PKCE; scope subset; token refresh; FILE_UPLOAD init and chunks; interrupted upload; status polling; inbox receipt on phone; manual final publish; public ID association; video list/query metrics; revoked token; 429/backoff; duplicate suppression; retention and disconnect deletion.

## Review preparation
Requested products: Login Kit (identify connected account), Content Posting API upload (send only approved creator content for user to finish in TikTok), video.list (show own public-video performance). No Direct Post, user.info.stats, profile extras, Research API or webhooks requested.

Demo must show real desktop app opening, connect and consent, content review and approval, inbox upload/status, TikTok finish flow and public-video metrics. Show all requested scopes with actual behavior; redact credentials and codes. TikTok allows up to 5 demo videos, 50 MB each. Do not claim tested functionality in review text before this matrix passes.

Production blockers: private/personal-use and unfinished apps are excluded by published review guidelines; current application is honestly described as authorized testing. Confirm product eligibility with TikTok before expecting production approval. A marketing website alone does not make an unfinished product production-ready. No final submission authorized without separate confirmation after a concrete demo and completed configuration are reviewable.

Migration: use approved production configuration and credentials, exact Desktop redirect and same least-privilege scopes; obtain fresh production user consent and tokens. Sandbox target-user grants/tokens do not automatically migrate. Do not assume replacing keys alone is sufficient. Keep test/production token stores separate.

## Official references
https://developers.tiktok.com/docs/en/login-kit-desktop
https://developers.tiktok.com/docs/en/content-posting-api-reference-upload-video
https://developers.tiktok.com/docs/en/content-posting-api-reference-get-video-status
https://developers.tiktok.com/docs/en/tiktok-api-v2-video-query
https://developers.tiktok.com/docs/en/add-a-sandbox
https://developers.tiktok.com/docs/en/app-review-guidelines
