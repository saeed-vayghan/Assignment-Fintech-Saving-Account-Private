
-- Spec 1 --
You are an expert software architect in Banking and fintech industry.
I want you to read intro-v2.md and understand the problem space. Then wait for my questions
-- End of Spec 1 --



-- Spec 2 --
Business requirements Summary section in non technical, assume you want to transform this section to clear technical items which are easy to udnerstand for an engineer or architect.

update this section and translate th requirements to clear technical items and lists.
- use simple english
- make sure all terms and abbreviations like PYI are also explained well
- Add a glossary sectiion and explain all terms like CAMT, matured, etc
- maybe some just need a basic english explanation, maybe some need to have a scenario to help engineers understand better
-- End of Spec 2 --


 

-- Spec 3 --
go through Legal requirements section and let me know how you can transform it to a better technical version
-- End of Spec 3 --




-- Spec 4 --
read software-architect-req.md to understand the scope of this architect who is support do this assignment to get hired and then wait for my command
-- End of Spec 4 --



-- Spec 5 --
intro-v2.md check and read Technical assignment 

we need to elaborate this section, do not add extra parts for now, just elaborate the current list and adding more technical details, it should help us to understand how we are going to design and implement the wholre project 
-- End of Spec 5 --



-- Spec 6 --
review the file and consider the below requirement:

In this assignment please consider the AWS Well-Architected best practices and consider topics such as monitoring, performance efficiency, security and reliability.
-- End of Spec 6 --



-- Spec 7 --
As you know:
AWS Well-Architected helps cloud architects build secure, high-performing, resilient, and efficient infrastructure for a variety of applications and workloads. Built around six pillars—operational excellence, security, reliability, performance efficiency, cost optimization, and sustainability

Review the intro-v2.md and think about aws well-architected framework and best practices. 
How can you update this document to be in line with aws well-architected framework and best practices?
-- End of Spec 7 --



-- Spec 8 --
we would like to go live with savings accounts in one year.
Read software-architect-req.md to know the tech stack of this project.
Suggest how many teams we need to build this project.
How many people in each team? what skills we need in each team?
Help me to create a roadmap for this project. How to break down this project into smaller milestones and phases.
what parts should we build first? what parts can we build later?
what parts can we build in parallel?
create a separate md file for this roadmap.
User MoSCoW framework to prioritize the features.

Adding more context to help you: The MoSCoW framework is a popular prioritization technique used in project management—particularly Agile—to categorize requirements, tasks, or features into four groups: Must have, Should have, Could have, and Won't have.
-- End of Spec 8 --



-- Spec 9 --
We need to make sure we are not missing even a single subtle point or item from the intro-v2.md file.
Go through intro-v2.md file and make sure we have covered all the requirements.
If you find any missing item, add it to the roadmap.

Then create a new file named `requirements-checklist.md`, add a checklist of all the requirements.
- Create a table aff all requirements, this table shold be easy to read and understand.
Add a column to the table to
    - indicate whether the requirement is covered or not.
    - indicate whether the priority of the requirement.
    - indicate whether tthe status of the requirement
    - indicate whether the requirement is covered or not.
    - what team is responsible for the requirement.
    - what phase it belongs to.
    - what category of MOSCOW it belongs to.
-- End of Spec 9 --



-- Spec 10 --
We have been working on this project for a while now, and we have a good understanding of the requirements and the tech stack. 

Lets imagign you are expert in "Software Architecture", "AWS well-architected framework", "Fintech", "Banking", "Poduct management", "Strategic decision making for the project", and "product vise lead (non-technical)".

review the project and md files and let me know what other important angles or aspects we should consider for this project.
-- End of Spec 10 --



-- Spec 11 --
Some added parts of requirements-checklist.md are beyound the technical/product scope of this assignment. 
I want to move the below sections to a new file named "beyound-assignment/checklist.md"
and remove moved items from the original files and then update requirements-checklist.md, roadmap.md, and teams.md  accordungly.

- RBAC for internal tools (compliance dashboard)
- Incident response playbooks (P1–P4, escalation, on-call)
- Post-incident review process (blameless retros)
- Reserved Instances / Savings Plans
- Graviton (ARM-based) instances
- Identify existing systems to modify
- Define new services to build from scratch
- Integration points with existing internal services
- Phased rollout plan (MVP vs full launch)
- Stakeholder: existing system constraints, preferences, timeline
- Deposit guarantee scheme reporting
- Withholding tax summaries to tax authority
- WCAG accessibility compliance for customer web app
- Customer SLA definitions (processing time commitments)
- Notification preferences: customer opt-in/opt-out for email/SMS
- Dormant account handling (flag inactive > 12m, unclaimed funds reporting)
- Onboarding funnel analytics: conversion tracking per step
- Currency rounding rules & day count conventions (e.g. Actual/365)
- Cross-region disaster recovery

in the checklist.md there some items with MoSCoW valuse as M(must) and s (should)
list them for yourself, and then make sure they are not present in requirements-checklist.md and roadmap.md becaue we have excluded all items in  checklist.md from the assignment.
-- End of Spec 11 --



-- Spec 12 --
what questions to ask for a system-design process, to let the architect know, here are some sample questions, give me a list of more questions that are relevant to the system design process of this project
- what is the number of potential users
- what is the number of potential transactions
- how to calculate the DB size, required storage, compute, etc
- What is the traffic volume?
-- End of Spec 12 --



-- Spec 13 --
04-db-selection.md we have a new file which needs to be updated.
review the project and explain what DB should be choose for what service, and why? add a list of reason why choosing any db for any service in this repo, add a secondary option for each one as well.
use simple english to explain your appraoch
-- End of Spec 13 --



-- Spec 14 --
Now lets focus on each team and its business boundaries:

For each team we need to know some details in order to drew the high-level diagram for each team:

- what is the purpose of this team
- if there is a UI, what is it, what is its purpose
- if it is a user-facing service or not
- is there any authentication or authorization needed for this team
- how many microservice this team need to have and what is the purpose of each microservice
- what DB is there, just type of db, name of, and purpose of DB
- is there any caching layer
- is there any meesage queue or any async communication needed for this team
- is there any external system this team need to interact with
- is there any third-party service this team need to interact with
- what other teams have interaction with this team, if there is any just mention the team name and purpose of interaction with as a single entity, do not go into details
- if this team creats any log and metrics, just mention them in a high-level way, nodetails yet
- does this team expose any API, what type of API


create a new file for each team in this directory: 03-system-design/06-teams
-- End of Spec 14 -- 


-- Spec 15 --
Think about edge-cases and all possible scenarios that any user request/operation or any other api call or process can get intrupted or cancelled or failed.

these edge cases could have some issues on user experience or this team businees

Categorize these problems into major and moderate and retryable

Major and Moderate should some how involve a human in the loop
retryable should do the same but if it reaches to a predefined threshold
-- End of Spec 15 --



-- Spec 16 --
- so now we know what DBs we need to have
- we also knwo what services/features are going to be implementd by what team
- we also know what tech stack we are going to use

What you need to do:

1. Implement some diagrams for each team that shows what services/features are going to be implementd by what team

2. Update diagrams to show how these team are going to interact with each other

3. Update diagrams to show how these team are going to interact with external systems

keep the design high-level, do not go into implementation details, we will think about data-models, schemas, data-flow, data providers, etc later.

add the diagrams to 03-system-design/07-teams-diagram.md
-- End of Spec 16 --
















- mention books as the reference
- using file-search project to search among private files



add OWASP Top 10 2023 to the security section of each team to the beyond-assignment/checklist.md file