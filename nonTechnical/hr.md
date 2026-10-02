
# Top 10 HR Questions

1. Tell me about yourself just an overview not in detail.
   Just tell the what tech stack u used in each company.

2. What are you looking up for in the next role.

   First, real ownership of systems.
   As, I'm looking for an environment where I can continue growing as a software engineer, especially around system design, distributed systems, and building reliable, scalable backend services.
   
   Second, a strong engineering team to grow with
   As, I value a collaborative team where engineers take ownership, communicate openly, and have the opportunity to contribute to technical decisions.
   
   On the backend side, I want to also go deeper into distributed systems.
   On the deployment side, I want to strengthen my platform and infrastructure skills.



3. Why do you want to join our company?
   
   You have to show interest that the company has its legacy and tell about the business and how it interest u to join that business. and interest in joining business can be like I have similar background of financial background (if the company is          financial one) or techstack is similar that aligns and maybe ownership is what u are looking forward.
   Can also contain points if mentioned in JD that I am looking for ownership and we have a techstack similarity.

5. Why are you interested in this role?
   
   Can also contain points if mentioned in JD that I am looking for ownership and we have a techstack similarity.

6. What I have been doing after May?
   
   My contract was for two years as it was a startup they got funding for two years from fraunhofer after it they have to let me go as they were strugling to find the investors.
   I took a little bit time off to visit my parents did arrangements for marrige traveled few places and now I am looking out for job.



## 1. Difficult situation / conflict
**Story: Video streaming architecture disagreement with CEO**

- **Situation:** I proposed using a queue between incoming video data, processing, and outgoing streaming.
- **Conflict:** The CEO was concerned that buffering would add unnecessary latency because frames were generated every ~33 ms.
- **My reasoning:** We were using UDP, so network bursts could still affect smoothness even if frame generation was consistent.
- **Action:** I explained the trade-off between **smoothness and latency** and discussed the concern openly.
- **Learning:** In technical disagreements, focus on trade-offs and evidence rather than trying to prove that my solution is right.

**Use for:** Conflict, disagreement, challenging an idea, communication.

---

## 2. Mistake / failure
**Story: Payment amount mismatch**

- **Situation:** We had a normal transaction flow and a retry flow. The backend stored amounts in cents/paise.
- **Problem:** The normal flow sent euros/rupees, while the retry flow was already sending cents/paise. The API contract was not explicit enough.
- **Impact:** The backend converted the retry amount again.
- **Example:** €10 → 1,000 cents. If the retry flow sent 1,000 cents and we converted again, it became 100,000 cents = €1,000.
- **Action:** We identified the mismatch and clarified the API contract for the amount unit.
- **Learning:** Verbal communication is not enough for important fields. API contracts should explicitly define units, formats, and examples, with tests covering all flows.

**Use for:** Mistake, failure, learning from failure, communication.

---

## 3. Unexpected problem / working under pressure
**Story: Client demo + 4G/5G issue**

- **Situation:** I had tested the drone system before a client demo, but on the demo day the drone did not start correctly.
- **Action:** I debugged the complete startup flow instead of assuming it was a software issue.
- **Root cause:** Our packets were around 1,400 bytes. When switching between 5G and 4G, some packets were not getting through correctly, so the required parameter download did not complete.
- **Result:** I identified the network-related root cause and explained why the system behaved differently from our test environment.
- **Learning:** For systems involving hardware and networks, testing must also consider environmental and network conditions.

**Use for:** Unexpected problem, pressure, debugging, problem solving.

---

## 4. Difficult technical decision
**Story: State design pattern**

- **Situation:** A full State design pattern would have provided a cleaner architecture.
- **Decision:** Because of a tight delivery timeline, I chose a simpler implementation instead of introducing additional abstraction.
- **Action:** I was aware of the trade-off and tested the state transitions thoroughly.
- **Learning:** The best engineering solution depends on complexity, maintainability, delivery time, and expected future changes.

**Use for:** Difficult technical decision, trade-offs, architecture.

---

## 5. Technical improvement / initiative
**Story: Circular buffer**

- **Situation:** The initial approach used a linear buffer to continuously store streaming data.
- **Problem:** A continuous stream needs predictable and bounded memory usage.
- **Action:** I proposed a circular buffer so that a fixed amount of memory could be reused.
- **Result:** The approach was better suited to continuous streaming and avoided unnecessary buffer growth.

**Use for:** Improvement, initiative, optimization, challenging an existing approach.

---

## 6. Helping a teammate
**Story: Junior developer + test cases**

- I led a project for writing test cases and worked with a junior developer.
- I delegated tasks and first explained the testing approach and coding style we wanted to follow.
- When he was blocked, I helped through short calls and explained the reasoning instead of simply giving the solution.
- This helped him become more independent while keeping the tests consistent.

**Key phrase:**  
> I helped him understand the reasoning rather than just giving him the solution.

---

## 7. Raising team quality
**Story: PayPal release automation**

- The release process had several manual steps.
- I worked on automating the release process using a release script.
- This made releases more repeatable and reduced manual effort and the chance of human error.
- It reinforced for me that team quality is also about improving engineering processes, not only writing better code.

**Use for:** Team quality, automation, process improvement.

---

## 8. Learning something hard quickly
**Story: Drone internet reconnection / AT commands**

- We needed more control over internet reconnection for the drone.
- I was not initially familiar with controlling the modem using AT commands.
- I researched the modem behaviour and relevant AT commands and implemented the required reconnection flow.
- This taught me how to quickly learn an unfamiliar technology and apply it to a real system.

**Use for:** Curiosity, learning quickly, unfamiliar technology.

---

## 9. Feedback / learning from failure
**Story: Fraud Management integration**

- **Situation:** I received a Fraud Management integration task on Wednesday with a Friday deadline because the system was planned to launch on Monday.
- **Problem:** It was my first time handling this integration, and under the time pressure my implementation was not at the architectural quality I wanted.
- **Action:** My team lead took over and completed the task. I then reviewed his PR carefully to understand his approach.
- **Improvement:** I identified that the integration could be made more generic using a suitable interface for supporting multiple payment gateways.
- I discussed the improvement with my team lead, and he agreed with the idea and praised the contribution.
- **Learning:** When something doesn't go as planned, I should study the better solution, understand why it is better, and apply that learning to future work.

**Use for:** Feedback, failure, growth, ownership, learning.

---

## 10. Cross-team collaboration
**Story: ACE event payload migration**

- I worked closely with other teams during the ACE event payload migration.
- I helped with the integration and supported other teams in adapting to the new payload.
- I received a collaboration award for my contribution.

**Use for:** Collaboration, cross-team communication, teamwork.

---

# Quick Story Map

| Question | Story |
|---|---|
| Difficult situation / conflict | **Video streaming architecture vs CEO** |
| Mistake / failure | **Payment amount mismatch** |
| Unexpected problem | **Client demo + 4G/5G** |
| Difficult technical decision | **State design pattern** |
| Technical improvement | **Circular buffer** |
| Helping a teammate | **Junior developer + test cases** |
| Raising team quality | **PayPal release automation** |
| Learning something hard quickly | **Drone + AT commands** |
| Feedback / growth | **Fraud Management integration** |
| Cross-team collaboration | **ACE payload migration** |

# Most Important Stories to Practice

1. **Video streaming architecture disagreement** — conflict
2. **Payment amount mismatch** — mistake/failure
3. **Fraud Management integration** — feedback/growth
4. **Client demo + 4G/5G** — problem solving under pressure
5. **Junior developer + testing** — teamwork




10. What are your salary expectations?
    
# HR Interview – Other most Important Questions

## About Yourself

1. Tell me about yourself.

2. Walk me through your resume.

3. Tell me about your current/recent role.

4. What are your main strengths?

5. What is one weakness or area you are working to improve?

---

## Motivation & Career

6. Why are you looking for a new job?

7. Why do you want to join our company?

8. Why are you interested in this role?

9. What are you looking for in your next job?

10. Where do you see yourself in 3–5 years?

11. What motivates you at work?

---

## Behavioral Questions

12. Tell me about a difficult situation you faced at work and how you handled it.

13. Tell me about a mistake you made and what you learned from it.

14. Tell me about a conflict with a colleague and how you resolved it.

15. Tell me about a time you received negative feedback.

16. Tell me about a time you had to work under pressure.

17. Tell me about a time you had to meet a tight deadline.

18. Tell me about a time you took ownership of something.

19. Tell me about a time you failed to achieve something. What did you learn?

20. Tell me about a time you disagreed with your manager or teammate.

---

## Working Style

21. Do you prefer working independently or in a team?

22. How do you prioritize your work?

23. How do you handle multiple tasks at the same time?

24. How do you handle ambiguity or unclear requirements?

25. How do you handle stress and pressure?

26. How do you respond to criticism or feedback?

27. What kind of manager do you work best with?

28. What kind of work environment do you prefer?

---

## Practical Questions

29. What are your salary expectations?

30. When can you start?

31. Are you interviewing with other companies?

32. Are you open to relocation/remote/hybrid work?

---

## Questions You Should Ask the Interviewer

33. What does the interview process look like from here?

34. What does success look like in this role during the first 3–6 months?

35. What are the biggest challenges someone in this role would face?

36. How would you describe the team and company culture?

37. What opportunities are there for learning and career growth?
