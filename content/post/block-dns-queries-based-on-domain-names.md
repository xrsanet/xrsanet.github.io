---
title: "Block DNS Queries Based on Domain Names via IPtables"
date: 2015-04-01T16:08:22Z
draft: false
categories: ["Sysadmin"]
---
This code will generate or apply rules for IPtables to block DNS queries based on domain names. I wrote this script because I did not want to manually convert each section of the domain into hex with their respective sizes every time I needed to block a random set of domains.

# Example

{{< highlight bash >}}
# ./BlockDomain.py 
Syntax: ./BlockDomain.py gen|apply <DOMAIN>...
  gen      : Generate the required IPTable rules
  apply    : Apply the required IPTable rules (Must be root)
 
Examples:
  ./BlockDomain.py gen www.zezelu.us
  ./BlockDomain.py gen www.3zezelu.com www.5zezelu.co.uk
  ./BlockDomain.py apply www.3zezelu.com www.5zezelu.co.uk
# ./BlockDomain.py  gen www.3zezelu.com www.5zezelu.co.uk
/sbin/iptables -I INPUT 1 -p udp --dport 53 -m string --algo bm --hex-string '|0377777707337a657a656c7503636f6d|' -j DROP
/sbin/iptables -I INPUT 1 -p udp --dport 53 -m string --algo bm --hex-string '|0377777707357a657a656c7502636f02756b|' -j DROP
#
{{< /highlight >}}

# Code

{{< highlight python >}}
#!/usr/bin/python
#
# Source: https://xrsa.net
# Author: Ben Draper
# Email: ben@xrsa.net
#
 
import binascii
import sys
import os
import subprocess
 
def f_usage(f_scriptname):
  """
  Shows Usage.
  """
  print "Syntax: %s gen|apply %s..." % (f_scriptname,"<DOMAIN>")
  print "  gen      : Generate the required IPTable rules"
  print "  apply    : Apply the required IPTable rules (Must be root)"
  print "\nExamples:"
  print "  %s gen www.zezelu.us" % (f_scriptname)
  print "  %s gen www.3zezelu.com www.5zezelu.co.uk" % (f_scriptname)
  print "  %s apply www.3zezelu.com www.5zezelu.co.uk" % (f_scriptname)
 
def fgen_iptable_rules(f_domain, f_type):
  """
  Converts or applies domain names to IPTABLE DROP rules.
  """
  section_content = ""
  for section in f_domain.split("."):
    section_content += (hex(len(section))[2:].zfill(2) + section.encode("hex"))
 
  if (f_type == "gen"):
    print "/sbin/iptables -I INPUT 1 -p udp --dport 53 -m string --algo bm --hex-string '|%s|' -j DROP" % (section_content)
 
  if (f_type == "apply"):
    if os.getuid() == 0:
      subprocess.call(["/sbin/iptables", "-I", "INPUT", "1", "-p", "udp", "--dport", "53", "-m", "string", "--algo", "bm",
      "--hex-string", "|"+section_content+"|", "-j", "DROP"])
    else:
      print "You must be root to change IPtables"
 
# START
if __name__ == "__main__":
  if ((len(sys.argv) > 2) and (sys.argv[1] == "gen" or sys.argv[1] == "apply")):
    for arg in sys.argv[2:]:
      fgen_iptable_rules(arg,sys.argv[1])
  else:
    f_usage(sys.argv[0])
{{< /highlight >}}