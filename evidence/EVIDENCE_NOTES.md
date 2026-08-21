Status of each file in this folder.

01-docker-manager-running.png
Real screenshot, Docker Desktop showing the wazuh-test container.

02-duplicate-rule-warning.txt
Still the actual terminal text, not a screenshot. This is the one gap left,
the moment the duplicate rule 5558 warning appeared on restart. Replace with
a real screenshot if one turns up.

03-logtest-before-threshold.png
Real screenshot. Rule 5503 matching on the seventh repeated PAM failure,
firedtimes '7', rule 5558 not yet triggered.

04-logtest-rule-5558-fired.png
Real screenshot, and the single most important piece of evidence in the
whole contribution. Rule 5558 firing on the eighth repeated PAM failure,
level 10, frequency 8, MITRE T1110.

05-pr-38270.png
Real screenshot of the open pull request.

06-pr-38154-closed.png
Real screenshot of the closed pull request, with the closing comment
explaining why.

07-rule-deployment-validation.png
Real screenshot, added after the rest. Shows a grep confirming rule 5558
appears exactly once in the deployed ruleset, plus wazuh-control status
confirming the Wazuh services were actually running before the logtest
results above are treated as meaningful.
