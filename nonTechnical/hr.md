
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

4. Why are you interested in this role?
   Can also contain points if mentioned in JD that I am looking for ownership and we have a techstack similarity.

5. What I have been doing after May?
   My contract was for two years as it was a startup they got funding for two years from fraunhofer after it they have to let me go as they were strugling to find the investors.
   I took a little bit time off to visit my parents did arrangements for marrige traveled few places and now I am looking out for job.


| Question                         | Story                         |
| -------------------------------- | ----------------------------- |
| Difficult situation/conflict     | **(e) Payment flow mismatch** |
| Failure/unexpected problem       | **(b) Demo + 4G/5G**          |
| Difficult technical decision     | **(d) State pattern**         |
| Technical improvement/initiative | **(c) Circular buffer**       |


6. Tell me about a difficult situation/conflict you faced at work.
   
   a) To consume the payload I can write our own custom class by understanding what response they are sending. Or I have can use the jar of other team to unload the response. So I was new to the company I was not aware of this situation.

   There was one problem when the internet was getting connected again we have to flush the video buffer and the script has to connect to internet using AT commands I fugure it how to do it earlier it was automtically how drone connects to internet again we were using this method so I researched about it how to do it faster.
    
   b) I have to give the demo to the client but on the demo day our code was not working even though I did everything what I could to avoid that situation I tested our code. Later on when I started debugging the issue I found out that it was thee        problem of 4G and 5G problem. as our data packet was of length 1400 bytes when we switched it to 4G few datapackets were not able to go through network and our drone didnot start as the downloading of parameter was not done the starting step.

   c) Circular buffer pattern was sugggested by me. I choose it earlier apporach was to use a linear buffer to store everything. 

   d) State design pattern - The correcct way to follow the state design pattern. but I choose to avoid it for faster proceeding of code. I tested thoroughly.

   e) There were two flows the how the txn goes one is the normal flow and other is the retry one. So the decision was to store the value in cents/paise and we expected the amount from frontend in euro and we were converting it into paise.
   But due to miscommunication as there were two people handling the project on frontend side on the retry flow the problem occured we were mistakenly receiving the amount on paise and then converting it into

   


8. Tell me about a mistake or failure and what you learned from it.
   a) There were two flows how the txn goes one is normal and other is to retry. The decision was to store the value in cents/paise at backend. I communicated verbally that we will be storing the amount in paise/cents at backend. I was expecting the money in euro/rupees from frontend in normal flow we were converting it into paise at our backend. But the problem occured when as the normal flow was sending the amount in euro/rupees but the retry flow was sending the amount in paise. So I was converting them again to paise. So if someone has paid 10 euro the amount and he goes through the retry flow he will be paying only 10 euros but in our system we were marking it as 1000 euro payed back. We defined the API contracts on our jira tickets.

9) Helping a teammate.
   When I was given the work to led the project of writting the test cases. I helped the junior developer by delegating the tasks understanding the concepts how the testcases need to be written by following a style that would suit best for the company. Helping him out where he was stuck by doing on call sessions.

10) raising team quality.
PayPal release Script to automate the release process. 

Analyzing the video streaming and capture the image if a person is available in the stream using th YOLO algorithm and using the other server for doing this. 


When I was designing the archithecture of Flybionic video streaming service. I was storing the service the data received in to the queue the processing it and then again storing the data into the queue and then sending it.
It was due to if I want to control the flow of outgoing for a smooth video I wanted to have it this way but CEO has the theory that the video coming to us is coming in smoothly as every video frame was generated after 33.33 ms so it is already smooth.
But my point was that we are using UDp still a bit of brust can be there due to network so smoothness will not be there as he didnot wanted to delay the frame but smoothness was a greater factor and it was not adding any substantial delay.

Fraud Management Integration task was given to me I was given the task on wednesday and I was expected to complete it till friday as monday we have to launch it.
Due to the pressure and first time I was handling the integration I was not able to do it like the archithecture wise. It was not up to the standards of software principles. Then my team leadd had to do it he picked up the task on Friday and completed it over the weekend.
I was very dishartened how things were. I took the PR of my team lead analyzed it how the task were done. While analyzing I found the the way integration was done can be improved. I setup a small 15 min meeting with the team lead proposed the design improvement
by creating the more generic like creating interface better suited for integrating both the Payment gateway. To be verry honest I vividly remember what I improved but I improved and he admittied it and also praised me in front of team to point out the design improvement in his PR.


When I was performing the ACE event Payload migration I received an award for collaboration as I was really helpful for the crossteam for integration for this event Payload migration.



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
