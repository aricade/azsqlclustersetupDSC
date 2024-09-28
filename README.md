# Azure SQL Windows Failover Cluster Setup DSC
Some DSC to be used in my Bicep Deployment to bootstrap a Windows Failover cluster & and SQL with an extension resource.

```
@batchSize(1)
resource resSetupCluster 'Microsoft.Compute/virtualMachines/extensions@2023-09-01' = [
  for (vm,i) in parVirtualMachines: if (parJoinDomain && parSetupCluster) {
    name: '${vm.name}/SetupCluster'
    location: parLocation
    properties: {
      publisher: 'Microsoft.Powershell'
      type: 'DSC'
      typeHandlerVersion: '2.77'
      autoUpgradeMinorVersion: true
      settings: {
        configuration: {
          url: uri(parStorageAccountEndpoint, 'scripts/SetupCluster.ps1.zip')
          script: 'SetupCluster.ps1'
          function: 'Cluster_Setup'
        }
        configurationArguments: {
          ClusterName: parClusterName
          FirstClusterNode: i==0 ? true : false
          ArrDisks: arrDSCDataDisks
          SQLSysAdminAccounts: parSqlClusterSettings.?SQLSysAdminAccounts ?? []
          SQLCollation: parSqlClusterSettings.?SQLCollation ?? 'SQL_Latin1_General_CP1_CI_AS'
          SQLTempDBDir: '${parSqlClusterSettings.DataDriveLetter}${parSqlClusterSettings.DBPath}'
          SQLTempLogDir: '${parSqlClusterSettings.LogDriveLetter}${parSqlClusterSettings.DBPath}'
          SQLUserDBDir: '${parSqlClusterSettings.DataDriveLetter}${parSqlClusterSettings.DBPath}'
          SQLUserDBLogDir: '${parSqlClusterSettings.LogDriveLetter}${parSqlClusterSettings.DBPath}'
          InstallSQLDataDir: '${parSqlClusterSettings.DataDriveLetter}${parSqlClusterSettings.DBPath}'
          SQLBackupDir: parSqlClusterSettings.?BackupDir ?? '${parSqlClusterSettings.DataDriveLetter}\\MSSQL\\Backup'
          SqlUpdateEnabled: parSqlClusterSettings.?UpdateEnabled ?? false
          ForceReboot: parSqlClusterSettings.?ForceReboot ?? false
          FailoverClusterNetworkName: parSqlClusterSettings.FailoverClusterNetworkName
          FailoverClusterIPAddresses: i==0 ? VM1FailoverClusterIPAddresses : VM2FailoverClusterIPAddresses
          SQLInstanceName: parSqlClusterSettings.?InstanceName ?? 'MSSQLSERVER'
          SQLInstanceId: parSqlClusterSettings.?InstanceId ?? 'MSSQLSERVER'
          SQLSourcePath: parSqlClusterSettings.?SQLSourcePath ?? '\\\\ott-file3.orbcomm.com\\deploy\\MSSQL\\SQLStandard2022'
          SqlFeatures: parSqlClusterSettings.?SqlFeatures ?? 'SQLENGINE,REPLICATION,FULLTEXT,DQ'
        }
      }
      protectedSettings: {
        configurationUrlSasToken: parStorageAccountSasToken
        configurationArguments: {
          ActiveDirectoryAdministratorCredential: {
            userName: parDomainJoinUserName
            password: parDomainJoinUserPassword
          }
          StorageAccountCredential:{
            userName: parStorageAccountCloudWitnessName
            password: i==0 ? storageAccountKey : 'NA'
          }
          SQLSvcCredential: {
            userName: parSqlClusterSettings.SQLSvcUserName
            password: parSQLSvcPassword
          }
		      AgtSvcCredential: {
            userName: parSqlClusterSettings.AgtSvcUserName
            password: parAgtSvcPassword
          }
          SqlSaCredential: {
            userName: 'sa'
            password: parSqlSaPassword
          }
        }
      }
    }
    dependsOn:[
      domainjoin
    ]
  }
```
