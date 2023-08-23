# mux

## What is it?

mux is a tool built on Tmux and Tmuxinator to orchestrate and automate testing of internal networks. 

Tmux can be used to run simultaneous terminal commands and organize testing under a single SSH session; Tmuxinator is a tool lets you automatically set up those windows and commands. 

Through an advanced configuration, it is possible to prepare and execute tools and commands in an exact manner- which would allow testers to focus on the interesting portions of their work and spend less time manually enumerating.

For testing sensitive systems or utilizing potentially dangerous tools (common examples are zerologon, Eternal Blue, and BlueKeep), we instead use mux to help organize and prepare the commands ahead of time. The tester remains in control the whole time!

Additionally, as a large history can be kept when organizing testing through Tmux, we have established automatic logging in order to avoid any potential loss of data. This is also highly beneficial in the instance that a client wishes to understand exactly what commands were ran, or in the occasion that a tester's access to a nomad will be cut off but they wish to review the exact steps of testing for reporting purposes.

[Tmux](https://github.com/tmux/tmux/wiki): A terminal tool used to concurrently run and switch between several programs in one terminal.
* Here is a cheatsheet for [Tmux commands](https://tmuxcheatsheet.com/)

[Tmuxinator](https://github.com/tmuxinator/tmuxinator): Tool used to create and manage Tmux sessions automatically.
* Tmuxinator uses YAML files to organize and create Tmux sessions

## Automated vs Manual

To help avoid surprises - below is a comprehensive list of what is ran automatically vs manually at various stages, and what steps need to happen before the start of each stage.

### InitScan:
* Note - this scan expects only two files: `/RSM/ipList.txt` and `/RSM/exclude.txt`. Both can be formatted as any typical nmap/masscan input file.
* Manual:
  * masscan, portsort, setting DC
* Automatic:
  * locating DC (/etc/resolv, dig)
  * validate nomad IP
  * preparing all other mux commands
  * install and configuring tmux, pipenv, dnsrecon, smbmap, docker, and msf db

### Unauthenticated:
* Note - this stage expects a masscan to have been ran and then the portsort utility having created a directory under `/RSM/scans/Lists` __AND__ for a dc or multiple DCs to have been set under `/RSM/dcIP.txt`.
* Manual:
  * nmap, asreproast, zerologon
* Automatic:
  * dnsrecon, anon ftp, snmp, IPMI ciphers, smb, enum4linux, coercAuth, bluekeep, eternalblue, ldap signing check, timeroasting

### Misc:
* Note: This step requires the Nessus license key as well as the masscan/portsort/dcIP.txt from the `InitScan`.
* Manual:
  * prepare command to validate credentials
* Automatic:
  * prepares msf, start gowitness, start nessus

### Authenticated:
* Note: Beyond the standard masscan/portsort/dcIP.txt, this section __REQUIRES__ valid domain credentials. Use the `ValidateCreds` window in `Misc` to verify that the credentials are correct- otherwise you may potentially lock out the account.
* Manual:
  * samTheAdmin exploit
  * auth'd ASREProast
* Automatic:
  * enumerate SMB shares (crackampexec, smbmap, dumpsterdiver)
  * enumerate LDAP (ldapdomaindump, getADUsers, ldap signing, get user desc, check MAQs, ldap-checker)
  * kerberoasting, findDelegations
  * samTheAdmin check
  * bloodhoud
  * ADCS (crackmapexec, certipy)
  * PetitPotam, PrintNightmare, GPP passwords

### LocalAdmin:
* Requires account credentials with local administrative privileges.
* Manual:
  * secretsdump, passTheHash, lsassy, DonPAPI

## Get started / Installation

MAKE SURE TO RUN THIS **_ONLY_** AFTER MOUNTING NOMAD.
```bash
wget https://raw.githubusercontent.com/JaredStemper/mux/main/nomadConfig.sh -O /RSM/nomadConfig.sh
/bin/bash /RSM/nomadConfig.sh
```

## Overview of files

tmuxinator - where the magic happens. Full guide will be included in separate word doc. main thing to remember is order (init-scan, unauthd, misc, authd, local-admin)

nomadConfig.sh - script to pull and organize all the files for this project into the nomad automatically (intended to be ran after ensuring nomad mounting is complete).

prefillTest.py - python script that grabs text and places it onto the command line so that the user can choose to modify it or more carefully track it's runtime.

tmux.conf - the default tmux configurations are somewhat lacking. This helps bridge the gap and adds a lot of power to tmux usage. (highly recommended to read through and understand all capabilities).

tmuxSessionHistoryCapture.sh - script used to periodically log all data currently found in the tmux server. This is especially useful when finishing a project and needing the ability to review every command that was ran once a nomad is disconnected from the client network.

## Learning Tmux

classic guide is [tmuxcheatsheet.com](tmuxcheatsheet.com).


Strong recommendation to read through the provided configuration file and understand what the various lines do.

Pro tips:
* The prefix key with default config is `Ctrl+b`
* Anytime a command is typed through the `prefix + :` command prompt, tab completion can be used if you don't recall the exact name of a command (e.g., `"kill-server"` can be found from tabbing `"kill"`) 
* `prefix + w`: view all panes, windows, and sessions. Use vim bindings or mouse to quickly switch (h j k l `+ enter`)
* Panes
  * `prefix + v`: split pane vertically
  * `prefix + s`: split pane horizontally
  * `prefix + ,`: rename pane
  * `prefix + enter`: cycle through all standard pane formatting (useful to quickly resize)
  * `prefix + {` or `prefix + }`: swap pane locations either right or left (useful in changing the pane you're focusing on without hiding the other pane) 
  * `prefix + z`: Zoom! used as a way to "fullscreen" a pane without saving that formatting. The active pane will fill the screen until you shift to another pane or press `prefix + z` again
  * `prefix + arrow key`: used to resize a pane slightly in the direction of the arrow key
  * `prefix  ctrl+arrow key` (two separate key strokes):  while holding the `ctrl` key, rapidly hitting the arrow key will more rapidly change the size of a pane
* Windows
  * `prefix + c`: new window
  * Use the `shift + arrow key` to move to other windows quickly
  * Use the mouse to click to other panes/windows and resize any panes
* Sessions
  * `prefix + (`: shift to next session (e.g., from initScan to unauth)
  * `prefix + )`: shift to prior session
  * `prefix + e`: set current session path to current pane path (useful if constantly in a different directory and wanting to open up new windows/panes in that new directory)
  * `prefix + d`: detach from current session. Now you will be back directly on the terminal and tmux will be running in the background
  * `tmux attach`: ran on the command line to re-attach to your most recent tmux session
  * `tmux kill-session`: ran on command line to kill the current session
* `tmux kill-server`: ran on command line to kill all tmux sessions (useful at end of assessment)
* Copy/Paste
  * Regular Clipboard
    * `shift + mouse` will highlight things you can use the classic ctrl+shift+c to copy/paste
  * Tmux Clipboard
    * Using your mouse to highlight text automatically copies whatever is highlighted to your Tmux clipboard
    * `prefix + [`: enter copy mode to more carefully copy items
      * Use vim key bindings to move cursor; use spacebar to start selection;
      * Use either `y` to copy and stay in copy mode (useful if in large text files) or `enter` to copy and exit copy mode
    * `prefix + ]`: paste the last item copied from copy mode
    * `prefix + =`: view all items copied in copy mode (useful to quickly paste various IPs/passphrases)

## Later TODO:

* Organizing all port/IP information through the Metasploit DB instead of text files
* Automated updates to Dojo test cases when commands have successfully ran
