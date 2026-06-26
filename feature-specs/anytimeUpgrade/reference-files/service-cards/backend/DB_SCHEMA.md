# Database Schema — bridge

> **This service owns no data store.**
>
> Bridge is a stateless BFF. It holds no tables, collections, or key-value
> entries of its own. All persistent domain data (customers, resellers,
> orders, subscriptions, pricelists) lives in the Adobe Partner API.
>
> The only in-process state is:
> - IMS access token cache (single module-level variable, lost on restart)
> - `/api/env` response cache (single module-level variable, 60s TTL)
>
> Neither constitutes an owned data store.
