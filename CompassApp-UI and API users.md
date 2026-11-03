# CompassApp-UI and API users

Reviewed against the current workspace on **2026-10-30**. This document describes code-supported behavior, not confirmation of deployed portals, enabled accounts, or live permission configuration.

## Application audiences

Compass is not restricted in code to Driven Solutions employees, agents, and compliance staff. It also supports client and agency accounts and backend services for external customer experiences. Account roles, login restrictions, client/agency associations, and route permissions must be considered separately.

| Audience | Verified support | What remains environment-specific |
| --- | --- | --- |
| Internal staff | Operational screens for membership, cases, queues, reports, and administration, governed by authentication and configured access rules | Which staff roles can use each page/action |
| Client staff | User types include `CLIENT`, `CLIENT_ADMIN`, and `CLIENT_USER`; user/profile models support client associations | Which accounts use the main Compass UI versus external sites, and their permitted pages |
| Client agency staff | `AGENCY_OWNER`, `AGENCY_MANAGER`, and `AGENCY_AGENT` have agency-selection logic; `CLIENT_ADMIN` has separate corporate-level agency-selection behavior | Actual agency assignments and authorization for each operation |
| Members/customers | Enrollment, payment-link, service-tracking, and survey flows backed by Compass APIs | Deployed external browser URLs and which flows are enabled |
| Integration and process accounts | `EXTERNAL_API` and `PROCESS` roles exist and are rejected by the main UI login handler | API credentials, permissions, scheduled-job deployment, and endpoint authentication configuration |

## Main Compass UI login and navigation

`/login/` renders the login page; `/user/login/` handles the custom login flow. That handler rejects accounts whose `UserProfile.is_external_user` is true and accounts with user type `PROCESS` or `EXTERNAL_API`.

It does not reject client user types solely because they are clients. A client account can therefore be eligible for main UI login when it is not marked external and satisfies authentication checks. This is not proof that any particular client currently signs in.

Examples of main UI paths are `/dashboard/`, `/search-member/`, `/member-enrollment/`, and `/queue/`. A path's name does not identify its intended audience: `/member-enrollment/` is not evidence of a member self-service login page.

Navigation uses permission-aware template checks. The permission middleware checks user-type and user-role page access for authenticated, non-AJAX requests. It skips anonymous and AJAX requests, so view-level authentication and authorization remain relevant. Feature permissions can impose additional action-specific restrictions. Do not infer complete endpoint access control from sidebar visibility or a role name.

Sources: [login handler](/home/projectds/projects/compass/usac/usac/views.py:708), [routes](/home/projectds/projects/compass/usac/usac/urls.py), [sidebar](/home/projectds/projects/compass/usac/usac/templates/usac/leftside-bar.html), [permission middleware](/home/projectds/projects/compass/usac/usac/middleware.py), and [page permissions](/home/projectds/projects/compass/usac/usac/code/business/page_permission.py).

## Client and agency associations

`User.clients` and `UserProfile.client` are separate associations. Agency users can be associated with agencies through `AgencyUser`. The `get_list_of_agencies()` helper treats agency roles and `CLIENT_ADMIN` differently; this helper alone does not establish every page's authorization or data scope.

Sources: [user models and agency selection](/home/projectds/projects/compass/usac/usac/models/user.py) and [people and access](architecture/people-and-access.md).

## External UI flows and Compass API paths

The following are backend route groups. They should not be presented as the browser URLs customers necessarily open. Payment, tracking, and survey links are constructed from configured portal URLs; this review did not verify those deployed sites.

| External flow | Compass backend route group | Purpose |
| --- | --- | --- |
| External enrollment | `/external_enrollment/...`, `/external_enrollment_api/...` | Endpoints used by external enrollment sites, including membership operations and SMS verification |
| Customer payments | `/api/payment_collector/...` | Link validation and payment operations for cases and membership terms |
| Service tracking | `/service_tracking/v1/...` | Tokenized retrieval of case/service-provider tracking details |
| Surveys | `/api/survey/v1/...` | Tokenized survey retrieval and submission |
| Vendor portal | `/api/v1/vendor/...` | Vendor profile, email-address, and phone-number API operations; separate from member access |

Authentication and permissions vary by endpoint; these route groups are not blanket grants of anonymous access. For example, membership-term payment validation and payment views explicitly use `AllowAny`, while validating the supplied payment-link token in their business logic.

`/login-external/` is an external-authentication/token endpoint, not a separate HTML login page. Its code checks the user's profile client against the requested client. Its current dispatch is also decorated with `login_required`; its name alone does not establish anonymous access.

Sources: [external routes and authentication view](/home/projectds/projects/compass/usac/usac/views.py:11438), [URL registration](/home/projectds/projects/compass/usac/usac/urls.py), [payment endpoints](/home/projectds/projects/compass/usac/usac/api/payment_collector/urls.py), [payment views](/home/projectds/projects/compass/usac/usac/api/payment_collector/views.py), [tracking endpoints](/home/projectds/projects/compass/usac/usac/api/service_tracking/urls.py), [survey endpoints](/home/projectds/projects/compass/usac/usac/api/survey/urls.py), and [vendor endpoints](/home/projectds/projects/compass/usac/usac/api/vendor_portal/urls.py).

## Members are not automatically login users

A covered person is represented by `MemberPerson`. Creating a membership/person does not by itself create an application login account. No general member login/dashboard in the main Compass UI was identified in this review. This does not rule out a separately deployed customer portal.

To determine actual usage, verify account types and external-user flags, client/agency associations, page/role/feature grants, deployed external frontend configuration, and endpoint-specific authentication. No account permissions or application behavior were changed as part of this documentation task.
