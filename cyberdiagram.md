
How to build the AI agent for Penetration with Claude Code

## 1. Overview - Overall Framework

Recently started experimenting with Claude Code to build an AI-based Penetration Testing tool.

Following the penetration testing workflow:

 reconnaissance → research → exploitation → post-exploit

Different modules are assigned to different agents to complete.

Brain completes the reconnaissance → research phase.

Executor completes the exploitation phase.

Finally, Brain completes the post-exploit phase.

## 2. Current Progress

The entire workflow has been completed, and successfully automated the penetration process of the first HTB box, automatically obtaining root privileges.

![](http://cloudiagram.com/images/blog/Pasted%20image%2020251227205602.png)

## 3. Overall Architecture Design


The system consists of two main components, or two agents:

One is the Brain agent (skill-agent), responsible for Reconnaissance & Intelligence Gathering, research, and finding vulnerability-corresponding POCs. Its primary work is to complete the target profile definition and develop the attack plan, including predicting target service ports and application attack level predictions.

The other is the Executor workflow, which receives the AttackPlan from Brain, performs assembly, executes specific exploit attempts, and has Exploit fallback & verification capabilities. After multiple attempts without breakthrough, it transfers back to Brain to develop custom exploits for final attempts.

![](http://cloudiagram.com/images/blog/Pasted%20image%2020251227202020.png)

The reason for this design is that Reconnaissance and Intelligence Gathering for the target is the most critical part, directly determining the Executor's success rate, so this work is assigned to Brain. This part also has the most interaction with LLM and consumes the most tokens.

Once the plan is complete, it's executed by Executor. There are two reasons for this: First, deterministic actions don't need multiple rounds of interaction with LLM consuming tokens. Second, actions that affect the external world, such as executing commands, should be implemented as much as possible without LLM to ensure the entire process is safe and controllable.

Finally, the success and failure records of the entire process are recorded, and Machine Learning is used to provide feedback training to Brain, improving Brain's prediction accuracy confidence in subsequent operations.

Overall architecture:

![](http://cloudiagram.com/images/blog/Pasted%20image%2020251227203353.png)



## 4. Brain Implementation


The Brain produces a `BrainIntelligence` package containing:

| Field                | Description                                              |
| -------------------- | -------------------------------------------------------- |
| `targetProfile`      | Target classification, security posture, technologies    |
| `targetIntelligence` | Detailed intelligence from profiler module               |
| `toolStrategy`       | Recommended tools and execution order                    |
| `discoveredServices` | Services found during reconnaissance                     |
| `vulnerabilities`    | Identified vulnerabilities with CVEs                     |
| `pocFindings`        | PoC database matches                                     |
| `attackVectors`      | Prioritized vectors with success probability & rationale |
| `confidence`         | Overall confidence score (0-100)                         |

The final attackPlan is determined through targetProfile, targetIntelligence, toolStrategy, discoveredServices, pocFindings, and attackVectors.

### targetProfile

Marks current target information, including network address, operating system, technology stack, etc.

![](http://cloudiagram.com/images/blog/Pasted%20image%2020251227203643.png)
### toolStrategy

Marks the tools to be selected, such as nmap, burpsuite, dirbuster and other MCP servers.

![](http://cloudiagram.com/images/blog/Pasted%20image%2020251227203711.png)


![](http://cloudiagram.com/images/blog/Pasted%20image%2020251227204032.png)


### vulnerabilities

Based on discoveredService and targetIntelligence information, search for vulnerabilities through exploit-db and CVE database.


![](http://cloudiagram.com/images/blog/Pasted%20image%2020251227203730.png)
### attackVectors

Determine current attack vectors, which are the exploit attempts we will hand over to Executor for execution.

![](http://cloudiagram.com/images/blog/Pasted%20image%2020251227203746.png)



![](http://cloudiagram.com/images/blog/Pasted%20image%2020251227205212.png)



## 5. Executor Implementation

**THE EXECUTOR (Workflow Model Agent)**:

- Assembly of attack plans from Brain's intelligence
- Execution of exploit attempts
- Fallback chain management
- Structured workflow operations

- **🧠 Brain (Skills-Based Agent)**: Cognitive tasks, reconnaissance, research, target profiling, attack vector planning
- **⚙️ Executor (Workflow Agent)**: Attack plan assembly, exploit execution, fallback chain management
- **Handoff Protocol**: Brain→Executor (BrainIntelligence), Executor→Brain (FallbackHandoff)
- **HITL Support**: `plan_only` mode stops at attack plan for human review

The Attack plan generated by Brain is handed over to Executor (Workflow Agent) via Handoff Protocol.

To execute the Handoff Attack plan.

There are 10 av-x vectors in total, in the following format:

```
{

"id": "av-3",

"name": "CVE-2007-2447 via netbios-ssn",

"targetService": {

"port": 139,

"protocol": "tcp",

"service": "netbios-ssn",

"priority": 110,

"software": "Samba",

"version": "3.0.20",

"banner": "Starting Nmap 7.93 ( https://nmap.org ) at 2025-12-27 12:19 +08\nNmap scan report for 10.10.10.3\nHost is up (0.043s latency).\n\nPORT STATE SERVICE VERSION\n139/tcp open netbios-ssn Samba smbd 3.0.20-Debian (workgroup: WORKGROUP)\n\nHost script results:\n|_smb2-time: Protocol negotiation failed (SMB2)\n| smb-security-mode: \n| account_used: <blank>\n| authentication_level: user\n| challenge_response: supported\n|_ message_signing: disabled (dangerous, but default)\n| smb-os-discovery: \n| OS:"

},

"vulnerability": {

"cveId": "CVE-2007-2447",

"name": "Samba 3.0.20 < 3.0.25rc3 - 'Username' map script' Command Execution (Metasploit)",

"service": {

"port": 139,

"protocol": "tcp",

"service": "netbios-ssn",

"priority": 110,

"software": "Samba",

"version": "3.0.20",

"banner": "Starting Nmap 7.93 ( https://nmap.org ) at 2025-12-27 12:19 +08\nNmap scan report for 10.10.10.3\nHost is up (0.043s latency).\n\nPORT STATE SERVICE VERSION\n139/tcp open netbios-ssn Samba smbd 3.0.20-Debian (workgroup: WORKGROUP)\n\nHost script results:\n|_smb2-time: Protocol negotiation failed (SMB2)\n| smb-security-mode: \n| account_used: <blank>\n| authentication_level: user\n| challenge_response: supported\n|_ message_signing: disabled (dangerous, but default)\n| smb-os-discovery: \n| OS:"

},

"severity": "critical",

"exploitType": "RCE",

"availableExploits": [

{

"id": "exploit-db-CVE-2007-2447",

"name": "Samba 3.0.20 < 3.0.25rc3 - 'Username' map script' Command Execution (Metasploit)",

"type": "metasploit",

"module": "exploit/multi/samba/usermap_script",

"scriptPath": "/opt/exploitdb/exploits/unix/remote/16320.rb",

"reliability": "great",

"expectedOutcome": "shell"

}

],

"validated": false,

"source": "exploit-db"

},

"priority": 100,

"successProbability": 0.75,

"approach": "Use Metasploit module: exploit/multi/samba/usermap_script",

"rationale": "critical severity vulnerability with great reliability exploit"

}


```

As you can see, the entire Attack plan contains information such as vulnerability, severity, exploitType, and availableExploits.

It is handed over to Executor to begin implementing the workflow, which includes a fallback chain mechanism to proceed to the next exploit attempt after failure.

![](http://cloudiagram.com/images/blog/Pasted%20image%2020251227205834.png)

## 6. Future Plans


### Knowledge Database

Add a Knowledge Database with multiple writeup markdown documents to optimize Brain's intelligence, increase attack vector accuracy, and improve confidence.

![](http://cloudiagram.com/images/blog/Pasted%20image%2020251227210444.png)


## 7. Important Notes


1. Always use command mode instead of MCP

2. The software installation address is outdated, search exploit

3. During the code writing process, there may be inconsistent variable naming. You need to inform the AI in advance whether to use camelCase or snake_case

4. The nmap service scan can only identify netbios-ssn and does not map to the SMB template name, causing the AI to fail to adjust this service vulnerability to a higher priority when calculating weights. This mapping needs to be added during the coding process.

```
Expected Result After Fix
For port 139 with netbios-ssn service:
Factor	Before	After
High-value service	0	+50
RCE-prone service	0	+30
Version detected	+20	+20
Common port (139)	+10	+10
Total Priority	30	110
CVE-2007-2447 will now be properly prioritized as a top attack vector.
User approved the plan

```

CVE-2007-2447 will now be properly prioritized as a top attack vector.

5. Sometimes when downloading installation packages, some software sources need to be changed to sudo apt update -o Acquire::Check-Valid-Until=false
But the Claude Code client
Must have the 

```

sudo apt update -o Acquire::Check-Valid-Until=false


```



6.For executing similar one-off commands, you need to install BUNDLE_GEMFILE locally in msfconsole

```
msfconsole -q -x "use exploit/unix/ftp/vsftpd_234_backdoor; set RHOSTS 10.10.10.3; set LHOST 10.10.14.8; set LPORT 4444; check; exit"
```

7. Python2 compatibility issue needs to be addressed, as many exploits are written in Python2, so Python2 compatibility needs to be considered
