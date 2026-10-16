Lightning API
=============

The Lightning API is the hosted cloud API behind InferencePort AI Lightning.
Its base URL is ``https://sharktide-lightning.hf.space`` and the generation
routes live under ``/gen``.

Authentication
--------------

Lightning accepts either of these bearer tokens:

* A Supabase access token for the signed-in user.
* A dashboard-generated Lightning API key.

Send both forms the same way:

.. code-block:: bash

   curl https://sharktide-lightning.hf.space/subscription \
     -H "Authorization: Bearer YOUR_TOKEN"

Generating API keys in the app
------------------------------

1. Sign in inside the desktop app.
2. Open ``Settings``.
3. Open the ``Account`` tab.
4. In ``Lightning API``, choose ``Generate API Key``.
5. Copy the key immediately. Only the unhashed secret is shown once.

The backing Supabase schema and RLS policies live in ``MISC/api.sql``.
Keys are stored hashed in ``public.lightning_api_keys`` and are scoped to the
owning user via ``auth.uid()`` RLS policies.

Headers
-------

Common request headers:

* ``Authorization: Bearer <supabase-jwt-or-api-key>`` for authenticated usage.
* ``X-Client-ID: <stable-client-id>`` to bind free-tier and usage tracking to a
  stable desktop/browser client.
* ``Accept: application/json`` for JSON endpoints.

Plan and usage endpoints
------------------------

``GET /subscription``
~~~~~~~~~~~~~~~~~~~~~

Returns the authenticated user's resolved plan and Stripe subscription view.

Response fields:

* ``email``
* ``signed_up``
* ``plan_key``
* ``plan_name``
* ``subscription``: ``null`` for free users, otherwise a list of active or
  recent subscription rows
* ``auth_type``: ``jwt`` or ``api_key``

``GET /usage``
~~~~~~~~~~~~~~

Returns the current usage snapshot for the resolved identity.

Response fields:

* ``plan_key``
* ``plan_name``
* ``generated_at``
* ``usage.cloudChatDaily``
* ``usage.imagesDaily``
* ``usage.videosDaily``
* ``usage.audioWeekly``

Each usage metric contains:

* ``limit``
* ``used``
* ``remaining``
* ``window``
* ``period``

``GET /tier-config``
~~~~~~~~~~~~~~~~~~~~

Returns the normalized plan catalog used by the app UI:

* ``defaultPlanKey``
* ``plans[]`` with ``key``, ``name``, ``url``, ``price``, ``limits``, and
  ``order``

``GET /tiers``
~~~~~~~~~~~~~~

Returns the paid plans only, mainly for upgrade UI.

Status and discovery endpoints
------------------------------

``GET /``
~~~~~~~~~

Permanent redirect to the public site.

``GET /models``
~~~~~~~~~~~~~~~

Scrapes the Ollama library page and returns public model metadata used by the
marketplace UI.

``GET /status``
~~~~~~~~~~~~~~~

Returns overall Lightning service health with:

* ``state``
* ``services``
* ``notifications``
* ``latest``

``HEAD /status/image``, ``/status/video``, ``/status/sfx``, ``/status/text``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Lightweight capability checks that return content-type headers only.

Generation endpoints
--------------------

All generation routes are rooted at ``/gen``.

``POST /gen/chat/completions``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

OpenAI-compatible chat completions endpoint.

Important request fields:

* ``messages``: required array
* ``stream``: optional boolean
* ``tools``: optional tool list
* ``tool_choice``: optional tool selection

Behavior notes:

* Lightning chooses an upstream model automatically based on prompt complexity,
  code signals, tools, and image presence.
* Authenticated requests consume ``cloudChatDaily`` usage from the resolved
  plan.
* Streaming responses are returned as ``text/event-stream``.
* The first streaming metadata chunk includes ``router_metadata.model_name``.

Example:

.. code-block:: bash

   curl https://sharktide-lightning.hf.space/gen/chat/completions \
     -H "Authorization: Bearer YOUR_API_KEY" \
     -H "Content-Type: application/json" \
     -d '{
       "messages": [
         {"role": "user", "content": "Write a haiku about shipping software."}
       ]
     }'

``POST /gen/image`` and ``GET /gen/image/{prompt}``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Image generation endpoint.

Request fields:

* ``prompt``: required
* ``mode``: optional, ``fantasy`` or ``realistic``
* ``image_urls``: optional list with up to two URLs or base64 image strings

Notes:

* Base64 image inputs are stored temporarily and re-served through
  ``/asset-cdn/assets/{image_id}``.
* Image generation consumes ``imagesDaily`` usage.

``POST /gen/video`` and ``GET /gen/video/{prompt}``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Pollinations-backed video generation.

Request fields:

* ``prompt``: required
* ``ratio``: optional, ``3:2``, ``2:3``, or ``1:1``
* ``mode``: optional, ``normal`` or ``fun``
* ``duration``: optional integer, clamped to ``1`` through ``10``
* ``image_urls``: optional list with up to two URLs or base64 image strings

Video generation consumes ``videosDaily`` usage.

``POST /gen/video/airforce`` and ``GET /gen/video/airforce/{prompt}``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Alternate Airforce-backed video endpoint with the same auth model. It also
consumes ``videosDaily`` usage.

``POST /gen/sfx`` and ``GET /gen/sfx/{prompt}``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Music and sound-effect generation. Requires ``prompt`` and consumes
``audioWeekly`` usage.

``POST /gen/tts`` and ``GET /gen/tts/{prompt}``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Text-to-speech generation. Requires ``prompt`` and also consumes
``audioWeekly`` usage.

``POST /gen/prompt_analyze``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Returns the router's chosen display model for a prompt payload. The request body
expects ``prompt`` as a message array.

Asset CDN
---------

``GET /asset-cdn/assets/{image_id}``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Returns temporary PNG files created when image or video generation receives
base64 image inputs.

Rate limits
-----------

Lightning currently tracks these metrics per resolved identity:

* ``cloudChatDaily``
* ``imagesDaily``
* ``videosDaily``
* ``audioWeekly``

If no bearer token is present, Lightning falls back to a free-tier identity
derived from ``X-Client-ID`` when available, otherwise from request IP and
user-agent.

Failure modes
-------------

Common status codes:

* ``400`` for invalid request payloads
* ``401`` for invalid, expired, or revoked bearer credentials
* ``413`` for prompts that exceed configured size limits
* ``429`` for plan rate-limit exhaustion
* ``500`` for upstream provider or server configuration failures

AI Shield
---------

AI Shield scores requests for abuse, fraud, AI-specific attacks, web
attacks, malicious files, weak passwords, marketplace scams and data
leaks, and enforces customer-defined rules. All routes live under
``/ai-shield``, use the same bearer tokens and count
against the plan's ``aiShieldDaily`` quota.

``POST /ai-shield/analyze``
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Full analysis of any mix of signals: ``email``, ``phone``, ``username``,
``ip``, ``device_fingerprint``, ``geolocation``, ``signup_time``,
``content`` / ``messages``, ``urls``, ``sender`` and a free-form
``metadata`` object (device, behaviour, login, payment, promo and trust
signals such as ``form_fill_ms``, ``failed_logins``,
``cards_tried_last_hour`` or ``account_age_days``).

Set ``task`` to choose a profile that re-weights categories and sets the
decision thresholds:

* ``general`` (default), ``signup``, ``login``, ``payment``
* ``content_moderation``, ``prompt_guard``, ``agent_tool_output``
* ``phishing``, ``dlp``
* ``api_request``, ``file_upload``, ``password``
* ``marketplace``, ``reviews``, ``domain``

Other inputs (also accepted by ``/scan``):

* ``request``: ``{method, url|path, query, headers, body}`` for WAF-style
  checks (SQL/NoSQL/command injection, XSS, template injection, XXE,
  Log4Shell, traversal, SSRF, scanner user agents, sensitive-path probes)
* ``file`` / ``files``: ``{name, content_base64|text, mime}`` up to 10 MB
  for static upload triage (real vs claimed type, executables, macros,
  scripted PDF/SVG/HTML, polyglots, zip-slip and zip bombs); extracted
  text also goes through the content detectors
* ``password``: strength, common/personal patterns and a breach count via
  Have I Been Pwned k-anonymity; never stored, echoed or sent to the LLM
* ``listing`` / ``review``: prohibited and counterfeit goods, marketplace
  scams, fake and incentivised reviews
* ``domain`` / ``domains`` and ``wallet``: look-alikes, risky or new
  domains, crypto address checksums

Optional ``config``: ``features`` (flag map), ``thresholds``
(``challenge`` / ``review`` / ``block``), ``category_weights``,
``trusted_domains``, ``redact`` and ``include_evidence``. For anything
Shield has no built-in detector for, add ``blocklists`` / ``allowlists``
(emails, email domains, domains, IPs with CIDR, usernames, phones,
countries, keywords, user agents, wallets, device fingerprints, account
IDs) and up to 50 ``custom_rules`` (``keywords`` or a ``pattern``,
``fields``, ``category``, ``weight``, ``action``). Unsafe regexes are
rejected and reported in ``warnings``.

Response fields:

* ``risk_score`` (0-100), ``confidence`` (0-1)
* ``decision``: ``allow``, ``challenge``, ``rate_limit``, ``review`` or ``block``
* ``reasons``, ``threat_categories``, ``category_scores``, ``actions``
* ``findings.sensitive_data`` and ``findings.urls``
* ``scores`` (deterministic score, hard-evidence floor, LLM review)
* ``redacted_content`` when redaction is on, ``evidence``, ``version``

``POST /ai-shield/scan``
~~~~~~~~~~~~~~~~~~~~~~~~

Content-only variant (``content``, ``text``, ``prompt``, ``messages`` or
``urls``) defaulting to ``task: prompt_guard``. Identity fields are
ignored.

``POST /ai-shield/detect-phishing``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Accepts ``sender``, ``reply_to``, ``subject``, ``content``, ``url`` /
``urls`` and optional ``authentication`` (``spf``, ``dkim``, ``dmarc``).
Returns ``is_phishing``, ``threat_level``, ``indicators``, ``explanation``
and the full verdict fields. It works without the LLM reviewer.

``GET /ai-shield/capabilities``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Public (no token): tasks, thresholds, feature flags with defaults,
categories and decisions.

``POST /ai-shield/check``
~~~~~~~~~~~~~~~~~~~~~~~~~

Single-purpose checks: ``type`` (``email``, ``domain``, ``url``, ``ip``, ``phone``, ``username``, ``user_agent``,
``wallet``, ``jwt``, ``hash``, ``secret``) plus ``value`` or up to 500 ``values``. Each result has a ``verdict``,
``risk_score``, matched ``rules`` and typed ``facts`` (for example ``facts.disposable`` for email). See :doc:`../ai-shield`
for the complete AI Shield guide.

From the command line, the same routes are available as
``ipx shield scan``, ``ipx shield analyze``, ``ipx shield phishing``,
``ipx shield url``, ``ipx shield request``, ``ipx shield file``,
``ipx shield password``, ``ipx shield listing``, ``ipx shield review``,
``ipx shield domain`` and ``ipx shield tasks``; ``--policy``, ``--block``
and ``--allow`` pass custom rules and lists. ``--fail-on <decision>``
exits with status 3 for use in CI.

Implementation notes
--------------------

Relevant files:

* ``lightning-api/app.py``
* ``lightning-api/gen.py``
* ``lightning-api/helper/ratelimit.py``
* ``lightning-api/helper/subscriptions.py``
* ``MISC/api.sql``
* AI Shield: ``src/shield/`` in the InferencePort-Lightning repository
