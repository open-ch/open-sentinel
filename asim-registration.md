# Register Open Systems ASIM parsers

This file is generated. The ARM template installs only Open Systems functions; it never edits Microsoft's built-ins or workspace-managed custom unifiers. Follow [Microsoft's parser-management guide](https://learn.microsoft.com/en-us/azure/sentinel/normalization-manage-parsers) to create the custom unifiers if needed.

Add each call below as one union member in the indicated workspace function, preserving other vendors' entries. Declare the referenced parameters in the workspace function with Microsoft's schema-specific types and defaults. Use named arguments: Open Systems preserves `disabled` as its third positional argument for compatibility. `pack` is not supported and must not be forwarded.

Register each Open Systems schema once per entry-point family. If the filtering custom unifier already calls `vim<Schema>Custom` or individual Open Systems parsers, replace that reference with the vendor union below instead of adding a duplicate. Existing bridges remain deployed as compatibility wrappers. The non-filtering wrappers take no parameters.

## AlertEvent

Filtering: add to `Im_AlertEventCustom` for `_Im_AlertEvent`:

```kusto
vimAlertEventOpenSystems(starttime=starttime, endtime=endtime, disabled=false, ipaddr_has_any_prefix=ipaddr_has_any_prefix, hostname_has_any=hostname_has_any, username_has_any=username_has_any, attacktactics_has_any=attacktactics_has_any, attacktechniques_has_any=attacktechniques_has_any, threatcategory_has_any=threatcategory_has_any, alertverdict_has_any=alertverdict_has_any, eventseverity_has_any=eventseverity_has_any)
```

Non-filtering: add to `ASim_AlertEventCustom` for `_ASim_AlertEvent`:

```kusto
ASimAlertEventOpenSystems()
```

## AuditEvent

Filtering: add to `Im_AuditEventCustom` for `_Im_AuditEvent`:

```kusto
vimAuditEventOpenSystems(starttime=starttime, endtime=endtime, disabled=false, srcipaddr_has_any_prefix=srcipaddr_has_any_prefix, actorusername_has_any=actorusername_has_any, operation_has_any=operation_has_any, eventtype_in=eventtype_in, eventresult=eventresult, object_has_any=object_has_any, newvalue_has_any=newvalue_has_any)
```

Non-filtering: add to `ASim_AuditEventCustom` for `_ASim_AuditEvent`:

```kusto
ASimAuditEventOpenSystems()
```

## Authentication

Filtering: add to `Im_AuthenticationCustom` for `_Im_Authentication`:

```kusto
vimAuthenticationOpenSystems(starttime=starttime, endtime=endtime, disabled=false, username_has_any=username_has_any, targetappname_has_any=targetappname_has_any, srcipaddr_has_any_prefix=srcipaddr_has_any_prefix, srchostname_has_any=srchostname_has_any, eventtype_in=eventtype_in, eventresultdetails_in=eventresultdetails_in, eventresult=eventresult)
```

Non-filtering: add to `ASim_AuthenticationCustom` for `_ASim_Authentication`:

```kusto
ASimAuthenticationOpenSystems()
```

## DhcpEvent

Filtering: add to `Im_DhcpEventCustom` for `_Im_DhcpEvent`:

```kusto
vimDhcpEventOpenSystems(starttime=starttime, endtime=endtime, disabled=false, srcipaddr_has_any_prefix=srcipaddr_has_any_prefix, srchostname_has_any=srchostname_has_any, srcusername_has_any=srcusername_has_any, eventresult=eventresult)
```

Non-filtering: add to `ASim_DhcpEventCustom` for `_ASim_DhcpEvent`:

```kusto
ASimDhcpEventOpenSystems()
```

## Dns

Filtering: add to `Im_DnsCustom` for `_Im_Dns`:

```kusto
vimDnsOpenSystems(starttime=starttime, endtime=endtime, disabled=false, srcipaddr=srcipaddr, domain_has_any=domain_has_any, responsecodename=responsecodename, response_has_ipv4=response_has_ipv4, response_has_any_prefix=response_has_any_prefix, eventtype=eventtype)
```

Non-filtering: add to `ASim_DnsCustom` for `_ASim_Dns`:

```kusto
ASimDnsOpenSystems()
```

## NetworkSession

Filtering: add to `Im_NetworkSessionCustom` for `_Im_NetworkSession`:

```kusto
vimNetworkSessionOpenSystems(starttime=starttime, endtime=endtime, disabled=false, srcipaddr_has_any_prefix=srcipaddr_has_any_prefix, dstipaddr_has_any_prefix=dstipaddr_has_any_prefix, ipaddr_has_any_prefix=ipaddr_has_any_prefix, dstportnumber=dstportnumber, hostname_has_any=hostname_has_any, dvcaction=dvcaction, eventresult=eventresult)
```

Non-filtering: add to `ASim_NetworkSessionCustom` for `_ASim_NetworkSession`:

```kusto
ASimNetworkSessionOpenSystems()
```

## WebSession

Filtering: add to `Im_WebSessionCustom` for `_Im_WebSession`:

```kusto
vimWebSessionOpenSystems(starttime=starttime, endtime=endtime, disabled=false, srcipaddr_has_any_prefix=srcipaddr_has_any_prefix, ipaddr_has_any_prefix=ipaddr_has_any_prefix, url_has_any=url_has_any, httpuseragent_has_any=httpuseragent_has_any, eventresultdetails_in=eventresultdetails_in, eventresult=eventresult, eventresultdetails_has_any=eventresultdetails_has_any)
```

Non-filtering: add to `ASim_WebSessionCustom` for `_ASim_WebSession`:

```kusto
ASimWebSessionOpenSystems()
```

After registration and forwarding activation, verify Open Systems results through both standard entry points and confirm other vendors' results are unchanged. A query returning no errors or no rows alone does not prove registration. See [the deployment and migration runbook](README.md).
