# Retrospective: Three Attempts, Three Lessons

Three failed attempts at the SCS, and each one exposed a layer I did not know was there. The lessons do not repeat, they go deeper. This file is the postmortem on each.

---

## Attempt 1, September 2025: I was studying for a test, not for understanding

I had done Maarek's course, Cantrill's deep dives, hundreds of practice questions. I knew the services. The exam still failed me, and not by a little.

The lesson was about the goal itself. I had been preparing to pass an exam, not to understand AWS security, and those are not the same thing. So I threw out the approach: diagrammed services, rebuilt from first principles, and started asking why things work the way they do instead of just what they do.

---

## Attempt 2, November 2025: Understanding on paper is not the same as building

This one hurt more, because I had done everything "right." I studied differently, went deeper, put in the work. I could explain KMS, diagram GuardDuty, walk through any architecture on a whiteboard.

Then the exam handed me scenarios built on Python automation, Terraform deployments, and SQL log analysis, tools I had read about but never touched. I knew *about* them. I had no idea how to actually *do* any of it. Knowing a thing and being able to build it are two different skills, and I only had the first one.

That is the failure that ended the reading phase and started the building phase: Python, boto3, Lambda, event-driven automation, and the three capstone security tools.

---

## Attempt 3, March 2026: Shallow building is not the same as deep building

My first crack at the new C03 format, and the strange part is that I walked away encouraged. I did not guess a single answer, I picked every one with intent, and the score report reflected it: close, with the gap concentrated well outside the areas I had built in.

The lesson lived in *where* I fell short. Theory was solid, and Lambda and automation were genuinely strong because I had poured real project time into them. Everything else was shallow. The C03 questions kept offering two or three answers that were all technically correct, where only one was the *most* correct given the qualifier: most cost efficient, least overhead, fastest, most secure. Discriminating between them takes the granular, worked-with-it knowledge you only get from building a service yourself, and I had that for a handful of services rather than all of them.

So the fix was not "study harder." It was "build wider, one service at a time," which became the full subdomain pass in [AWS Journeys Edge](https://github.com/Snowblind019/AWS-Journeys-Edge).

---

## What carries forward

Each fix is visible in the repos rather than just stated here. Attempt one moved me from memorization to first principles. Attempt two moved me from theory to code. Attempt three moved me from narrow depth to depth across every subdomain of the blueprint. Attempt four is the test of whether that third fix holds.
