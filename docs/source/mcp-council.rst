Council MCP Server
==================

The Council MCP server lets any MCP-compatible assistant (Claude, Claude Code,
Cursor, VS Code, or your own agent) ask a **council of LLMs** a hard question.
One tool call sends your problem to several models in parallel, has them review
each other's answers for mistakes, and then has judge models synthesize a single
final answer.

You can set it up from the **MCP (Council)** tab in the
`InferencePort console <https://inference.js.org/console/>`_. The tab shows the
server URL, copy-ready configuration for popular clients, which wallet will be
billed, and a connection test.

Server details
--------------

* **URL:** ``https://api.inferenceport.ai/mcp/council/`` (keep the
  trailing slash)
* **Transport:** Streamable HTTP
* **Authentication:** OAuth sign-in, or an InferencePort API key sent as a
  Bearer token

Tools
-----

``council_reason``
   Run the council on a problem.

   * ``problem``: the question to solve. Be specific.
   * ``tier``: which council to use (default ``basic``).
   * ``iteration_rounds``: 1 to 3 review rounds (default 1).
   * ``custom_models`` / ``custom_judges``: optional lists of InferencePort
     model IDs that override the tier's models or judges.

``council_list_tiers``
   List the tiers with their model counts, judge counts and surcharges.

``council_list_models``
   Show the council, judge and fallback models of one tier.

Higher tiers use more and stronger models and cost more. Call
``council_list_tiers`` for the current surcharges, because they can change.

Authentication
--------------

Pick whichever fits your client.

OAuth sign-in (recommended)
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Add the server URL to a client that supports MCP authorization. The client opens
a browser window, you sign in to InferencePort and approve access, and nothing
needs to be copied or stored. Approved clients are listed under **Authorized
Applications** in the console, where you can revoke them at any time.

API key
~~~~~~~

1. In the console, open **API key** (or **API keys** inside an organization)
   and generate a key. It starts with ``ipa_live_``.
2. Configure your client to send ``Authorization: Bearer ipa_live_...``.

Keep keys secret and never commit them to a repository.

Client setup
------------

Claude Code, OAuth:

.. code-block:: bash

   claude mcp add --transport http council https://api.inferenceport.ai/mcp/council/
   # then run /mcp inside Claude Code and choose "council" to sign in

Claude Code, API key:

.. code-block:: bash

   claude mcp add --transport http council https://api.inferenceport.ai/mcp/council/ \
     --header "Authorization: Bearer ipa_live_YOUR_KEY"

Cursor and other clients that read ``mcpServers`` (omit ``headers`` to use
OAuth):

.. code-block:: json

   {
     "mcpServers": {
       "council": {
         "url": "https://api.inferenceport.ai/mcp/council/",
         "headers": { "Authorization": "Bearer ipa_live_YOUR_KEY" }
       }
     }
   }

VS Code (``.vscode/mcp.json``):

.. code-block:: json

   {
     "servers": {
       "council": {
         "type": "http",
         "url": "https://api.inferenceport.ai/mcp/council/",
         "headers": { "Authorization": "Bearer ipa_live_YOUR_KEY" }
       }
     }
   }

Claude Desktop (through the ``mcp-remote`` bridge):

.. code-block:: json

   {
     "mcpServers": {
       "council": {
         "command": "npx",
         "args": [
           "-y", "mcp-remote",
           "https://api.inferenceport.ai/mcp/council/",
           "--header", "Authorization:Bearer ipa_live_YOUR_KEY"
         ]
       }
     }
   }

Billing
-------

Every model and judge call in a council run is billed per token, exactly like a
normal chat request. A flat council surcharge is added on top. Before any model
is called, the server checks that your wallet can cover the surcharge. If every
council member fails, the surcharge is not charged.

Limits apply to protect your balance: problem length, a cap on custom models
and judges, a total run timeout, and a cap on council runs in progress at once
per user.

Organizations
-------------

Council calls can be billed to an organization instead of your personal wallet.
The organization's model and modality policies, spend limits, IP allowlist and
API key restrictions apply to every call. Your role must be allowed to spend the
organization's credits (owner, admin or member).

* **Organization API key:** the key already belongs to its organization, so
  council calls bill that organization automatically. No extra header is needed.
  Create one under **API keys** in the organization's console section.
* **OAuth sign-in:** add the header ``X-InferencePort-Org: <organization-slug>``
  to your client configuration. Without it you are billed personally. If you are
  not a member of that organization the call fails with a clear error and
  nothing is charged. Clients that cannot send custom headers should use an
  organization API key instead.

The console snippets add this header for you while an organization is active in
the context switcher.

Troubleshooting
---------------

``401`` or "invalid token"
   The sign-in expired or the key is wrong or revoked. Reconnect the client, or
   generate a new key.

"Insufficient credits"
   The wallet that is being billed cannot cover the surcharge. Add credits, or
   check which wallet you are billing (personal or organization).

"You are not a member of that organization"
   The ``X-InferencePort-Org`` slug is wrong, or your account is not in that
   organization.

"Your role ... cannot bill usage to this organization"
   Auditors and billing managers cannot spend credits. Ask an owner or admin.

"Organization monthly spend limit reached"
   The organization or your personal budget in it is used up for the month.

"The council did not finish within ... seconds"
   Use a lower tier or fewer ``iteration_rounds``.

See Also
--------

* :doc:`integrations`
* :doc:`developer/lightning-api`
