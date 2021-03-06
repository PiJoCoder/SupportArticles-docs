---
title: Troubleshooting automatic failover problems
description: This article provides the troubleshooting steps for the problems that occur during automatic failover in SQL Server.
ms.date: 11/05/2020
ms.prod-support-area-path: Availability Groups
ms.reviewer: ramakoni, cmathews
ms.prod: sql
---
# Troubleshooting automatic failover problems in SQL Server 2012 AlwaysOn environments

This article helps you resolve the problems that occur during automatic failover in SQL Server.

_Original product version:_ &nbsp; SQL Server  
_Original KB number:_ &nbsp; 2833707

## Summary  

Microsoft SQL Server 2012 AlwaysOn availability groups can be configured for automatic failover. Therefore, if a health issue is detected on the instance of SQL Server that is hosting the primary replica, the primary role can be transitioned to the automatic failover partner (secondary replica). However, the secondary replica cannot always be transitioned to the primary role, instead being transitioned only to the resolving role. Unless the primary replica returns to a healthy state, there is no replica in the primary role. Additionally, the availability databases are inaccessible.

This article lists some common causes of unsuccessful automatic failover. Additionally, this article discusses the steps that you can perform in order to diagnose the cause of these failures.

## The symptoms when an automatic failover is triggered successfully

When an automatic failover is triggered on the instance of SQL Server that is hosting the primary replica, the secondary replica transitions to the resolving role and then to the primary role. Additionally, you receive error messages in the SQL Server log report that resemble the following:

> The state of the local availability replica in availability group '\<Group name>' has changed from 'RESOLVING_NORMAL' to 'PRIMARY_PENDING'  
The state of the local availability replica in availability group '\<Group name>' has changed from 'PRIMARY_PENDING' to 'PRIMARY_NORMAL'

![Error log](./media/troubleshooting-automatic-failover-problems/error-log-notepad.png)

> [!NOTE]
> The secondary replica transitions successfully from a **RESOLVING_NORMAL** status to a **PRIMARY_NORMAL** status.

## The symptoms when automatic failover is unsuccessful

If an automatic failover event is not successful, the secondary replica does not successfully transition to the primary role. Therefore, the availability replica will report that this replica is in Resolving status. Additionally, the availability databases report that they are in Not Synchronizing status, and applications cannot access these databases.

For example, in the following image, SQL Server Management Studio reports that the secondary replica is in Resolving status because the automatic failover process was unable to transition the secondary replica into the primary role:

![Availability Replicas](./media/troubleshooting-automatic-failover-problems/availability-replicas.png)

This article describes several possible reasons that automatic failover may not succeed, and how to diagnose each cause.  

## Case 1: "Maximum Failures in the Specified Period" value is exhausted

The availability group has Windows cluster resource properties, such as the Maximum Failures in the Specified Period property. This property is used to avoid the indefinite movement of a clustered resource when multiple node failures occur.

To investigate and diagnose whether this is the cause of unsuccessful failover, review the Windows cluster log (Cluster.log), and then check the property. To do this, follow these steps:

- Step 1: Review the data in the Windows cluster log (Cluster.log)

   1. Use Windows PowerShell to generate the Windows cluster log on the cluster node that is hosting the primary replica. To do this, run the following cmdlet in an elevated PowerShell window on the instance of SQL Server that is hosting the primary replica:

      ```powershell
      Get-ClusterLog -Node <SQL Server node name> -TimeSpan 15
      ```

      ![Windows PowerShell](./media/troubleshooting-automatic-failover-problems/windows-powershell-image.png)

      > [!NOTE]
      >
      > - The -TimeSpan 15 parameter in this step is used under the assumption that the issue being diagnosed occurred in the previous 15 minutes.
      > - By default, the log file is created in %WINDIR%\cluster\reports.

   2. Open the *Cluster.log* file in Notepad in order to review the Windows cluster log.

   3. Click **Edit** in Notepad, and then click **Find**, and search for the string "failoverCount" at the end of the file. Review the results, and you should find a message that resembles the following:

      > Not failing over group \<Resource name>, failoverCount 3, failoverThresholdSetting \<Number>, computedFailoverThreshold 2

      ![Cluster Notepad](./media/troubleshooting-automatic-failover-problems/cluster-notepad-image.png)

- Step 2: Check the Maximum Failures in the Specified Period property

   1. Start Failover Cluster Manager.
   2. In the navigation pane, click **Roles**.
   3. In the **Roles** pane, right-click the clustered resource, and then click **Properties**.
   4. Click the **Failover** tab, and check the **Maximum Failures in the Specified Period** value.

      ![Maximum Failures in the Specified Period](./media/troubleshooting-automatic-failover-problems/properties-image.png)

      > [!NOTE]
      > The default behavior specifies that if the clustered resource fails three times in a six hour period, it should remain in the failed state. For an availability group, this means the replica is left in the RESOLVING state.

Conclusion

After you analyze the log, you find that the **failoverCount** value of 3 is greater than the **computedFailoverThreshold** value of 2. Therefore, Windows cluster cannot complete the failover operation of the availability group resource to the failover partner.

Resolution

To resolve this issue, increase the **Maximum Failures in the Specified Period** value.

> [!NOTE]
> Increasing this value may not resolve the issue. There may be a more critical issue that causes the availability group to fail many times in a short period, which is 15 minutes by default. Increasing this value may only cause the availability group to fail more times before remaining in a failed state. We recommend an aggressive troubleshooting effort to determine why automatic failover keeps occurring.  

## Case 2: Insufficient NT Authority\SYSTEM account permissions

The SQL Server Database Engine resource DLL connects to the instance of SQL Server that is hosting the primary replica by using ODBC in order to monitor health. The logon credentials that are used for this connection are the local SQL Server NT AUTHORITY\SYSTEM login account. By default, this local login account is granted the following permissions:

- **Alter Any Availability Group**  
- **Connect SQL**  
- **View server state**

If the NT AUTHORITY\SYSTEM login account lacks any of these permissions on the automatic failover partner (the secondary replica), then SQL Server cannot start health detection when an automatic failover occurs. Therefore, the secondary replica cannot transition to the primary role. To investigate and diagnose whether this is the cause, review the Windows cluster log. To do this, follow these steps:

   1. Use Windows PowerShell to generate the Windows cluster log on the cluster node. To do this, run the following cmdlet in an elevated PowerShell window on the instance of SQL Server that is hosting the secondary replica that did not transition into the primary role:

      ```powershell
      Get-ClusterLog -Node <SQL Server node name> -TimeSpan 15
      ```

      ![PowerShell window](./media/troubleshooting-automatic-failover-problems/windows-powershell-image.png)

   2. Open the Cluster.log file in Notepad in order to review the Windows cluster log.
   3. You can find error messages that resemble the following:

      > Failed to run diagnostics command.
      The user does not have permission to perform this action.

      ![Error messages](./media/troubleshooting-automatic-failover-problems/error-messages-image.png)

Conclusion

The Cluster.log file reports that there is a permission issue when SQL Server runs the diagnostics command. In this example, the failure was caused by removing the View server state permission from the NT AUTHORITY\SYSTEM login account on the instance of SQL Server that is hosting the secondary replica of an automatic failover pair.

Resolution

To resolve this issue, grant sufficient permissions to the NT AUTHORITY\SYSTEM login account for the health detection of SQL Server Database Engine resource DLL.  

## Case 3: The availability databases are not in a SYNCHRONIZED state

In order to automatically fail over, all availability databases that are defined in the availability group must be in a SYNCHRONIZED state between the primary replica and the secondary replica. When an automatic failover occurs, this synchronization condition must be met in order to make sure that there is no data loss. Therefore, if one availability database in the availability group in the synchronizing or not synchronized state, automatic failover will not successfully transition the secondary replica into the primary role.

For more information about the required conditions for an automatic failover, see the **Conditions required for an Automatic Failover** section and the **Synchronous-commit replicas support two settings** section of the article: [Failover and Failover Modes (Always On Availability Groups)](/sql/database-engine/availability-groups/windows/failover-and-failover-modes-always-on-availability-groups).

To investigate and diagnose whether this is the cause of unsuccessful failover, review the SQL Server error log. You should find an error message that resembles the following:

One or more databases are not synchronized or have not joined the availability group

![ERRORLOG Notepad Image](./media/troubleshooting-automatic-failover-problems/error-log-image.png)

To check whether the availability databases were in the SYNCHRONIZED status, follow these steps:

1. Connect to the secondary replica.
2. Run the following SQL script in order to check the `is_failover_ready` value for all availability databases in the availability group that did not fail over.

   > [!NOTE]
   > A value of zero for any of the availability databases can prevent automatic failover, and this value indicates that the availability database was not SYNCHRONIZED.

    ```sql
    Select database_name, is_failover_ready from sys.dm_hadr_database_replica_cluster_states where replica_id in (select replica_id from sys.dm_hadr_availability_replica_states)
    ```

   ![Query Image](./media/troubleshooting-automatic-failover-problems/sql-query.png)

Conclusion

A successful automatic failover of the availability group requires all availability databases be in the SYNCHRONIZED status. For more information about availability modes, see [Differences between availability modes for an Always On availability group](/sql/database-engine/availability-groups/windows/availability-modes-always-on-availability-groups).

