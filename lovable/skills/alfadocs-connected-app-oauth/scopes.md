# AlfaDocs OAuth scopes

Scopes are a single **space-separated** `scope` parameter (not a repeated parameter). The end user sees every scope you request on the consent screen, so request only what the app actually uses. Grant types are `authorization_code` + `refresh_token`; token auth is `client_secret_post`.

## Catalogue

| Area | Scopes |
|---|---|
| Patients | `patient:view` `patient:list` `patient:create` |
| Patient phone numbers | `patient:phone-number:view` `patient:phone-number:list` `patient:phone-number:create` |
| Appointments | `appointment:view` `appointment:list` `appointment:create` `appointment:booking-reasons:list` `appointment:care-plan-entry:list` |
| Billing | `bill:view` `bill:list` `bill-row:list` `bill-liability:list` `account:list` `payment-type:list` |
| Care plans | `care-plan:view` `care-plan:list` `care-plan:signature-status:view` `care-plan:care-plan-entry:view` `care-plan-entry:list` `care:view` |
| Custom fields | `custom-field:list` `custom-field:create` |
| Operators | `operator:list` `operator:view` |
| Scheduling | `free-slot:list` `chair:list` `color:list` `specialty:list` `care-category:list` |
| App integration | `app:integration:setStatus` `app:ui-element:update` `subscription:view-marketplace-app` |
| Webhooks | `webhook:appointment` `webhook:patient` `webhook:care-plan` `webhook:care-plan-entry` `webhook:bill-liability` `webhook:payment-request` |
| Notes | `note:create` `note:update` |
| Misc | `callback:payment` `archive:view` |

## Example — minimal recall app

An app that reads patients and their last appointment to surface overdue recalls needs only:

```
patient:view patient:list patient:phone-number:view patient:phone-number:list appointment:view appointment:list operator:list
```

Add `webhook:appointment webhook:patient` if you want push updates instead of polling.

## Notes

- Picking the right scopes is part of provisioning the OAuth client (see SKILL.md Step 1). If you add scopes later, the client config must be updated by an AlfaDocs admin and users may need to re-consent.
- Don't request `*:create` / `*:update` / webhook scopes unless the app genuinely writes or subscribes — read scopes are far easier to get approved and trusted.
