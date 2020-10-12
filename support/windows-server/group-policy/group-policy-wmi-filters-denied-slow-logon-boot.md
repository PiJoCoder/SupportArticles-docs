---
title: Group policy with WMI filters can be denied or cause slow logon/boot
description: Provides the resolution for the issue that Group policy with WMI filters can be denied or cause slow logon/boot
ms.date: 09/14/2020
author: Deland-Han
ms.author: delhan 
manager: dscontentpm
audience: itpro
ms.topic: troubleshooting
ms.prod: windows-server
localization_priority: medium
ms.reviewer: kaushika, kajain
ms.prod-support-area-path: Problems applying Group Policy objects to users or computers
ms.technology: GroupPolicy
---
# Group policy with WMI filters can be denied or cause slow logon/boot

This article provides the resolution for the issue that Group policy with WMI filters can be denied or cause slow logon/boot.

_Original product version:_ &nbsp; Windows Server 2012 R2  
_Original KB number:_ &nbsp; 2663850

## Symptoms

Group policy with WMI filter fails to apply. The WMI query (WMI filter linked to a GPO) times out due to which the GPO does not apply to the computers, intermittently.

GPSVC logs show the following:

GPSVC(320.af8) 12:00:14:840 FilterCheck: Evaluate returned error. hr=0x80041069  
GPSVC(320.af8) 12:00:14:840 ProcessGPO: The GPO does not pass the filter check and so will not be applied.  

ERR for 0x80041069 gives the following result:

>\# for hex 0x80041069 / decimal -2147217303  
> WBEM_E_TIMED_OUT wbemcli.h  
  \# as an HRESULT: Severity: FAILURE (1), FACILITY_ITF (0x4), Code 0x1069  
  \# for hex 0x1069 / decimal 4201  
   ERROR_WMI_INSTANCE_NOT_FOUND winerror.h  
  \# The instance name passed was not recognized as valid by a
  \# WMI data provider  
  \# 2 matches found for "0x80041069"

## Cause

WMI filtering was taking time while being processed. It was trying to enumerate all the domain names as query used was:

[ Select * from Win32_NTDomain where ClientSiteName  = "XYZ" ]

## Resolution

Add domain name component to the WMI query to resolve the issue (along with the AND operator): 

Select * from Win32_NTDomain where DomainName = "Domain_name" AND ClientSiteName  = "XYZ"

Make sure the below hotfix is already installed:

After you apply a WMI filter, the GPO does not take effect on a client computer that is running Windows 7 or Windows Server 2008 R2
 [https://support.microsoft.com/kb/979383](https://support.microsoft.com/help/979383) 

## More information

Timeout for WMI queries:

Per-Vista = None  
Vista onwards = 30 seconds (hardcoded)

Behavioral change when using WMI-Filter in a GPO  
 [https://support.microsoft.com/kb/2587435](https://support.microsoft.com/help/2587435) 

In one of the cases, it was informed that removing customized security filtering from the GPO (that has WMI filter configured) resolved the issue, however, it has not been tested or verified.