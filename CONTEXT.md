# Fail Verification

Decides why an automated test failed, so that daily failures from the Robot Framework suites (Selenium web, later Appium mobile) are sorted into causes with evidence and a next action, without a human triaging each one.

## Language

### Grouping

**Failure group**:
A set of failed tests from one run that share the same feature, platform (PC or phone) and full error message, and are therefore assumed to share one root cause.
_Avoid_: Cluster, bucket, batch

**Candidate**:
The one failed test picked from a failure group to be analysed; its verdict stands for the whole group.
_Avoid_: Representative, sample, pick

**Platform**:
The kind of client a test drives: PC (web, via Selenium) or phone (mobile app, via Appium).
_Avoid_: Device, channel
