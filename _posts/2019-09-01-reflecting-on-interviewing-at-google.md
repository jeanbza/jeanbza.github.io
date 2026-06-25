---
layout: post
title:  "Reflecting on interviewing at Google"
date:   2019-09-01 15:55:23 -0600
categories: 
toc: true
---

Disclaimer: This document makes statements and gives advice about interviewing
at Google. This is _not at all_ an official accounting of what it's like to
interview at Google. This is entirely my own interpretation, likely riddled with
inaccuracies to the extent that a Googler in recruitment would probably develop
a migraine reading this. If you'd like the official word, check out
careers.google.com or talk to a recruiter.

Foreword: SWE is an abbreviation for Software Engineer and is often pronounced
"swee".

# Aiming high

For me, the ultimate goal has always been Google.

I began programming during Ms. Roszko's Computer Science I course in my
sophomore year of high school. I graduated university seven years later with
degrees in Math and Computer Science. Sometime between then, I had convinced
myself that the pinnacle of a career in computer science was a job at Google.

I'm not sure why. Perhaps the infamously hard interview process, which seemed to
accept only the best of the best. Perhaps the reputably sky-high salaries. Or,
most likely it was that the work was that special kind of interesting and
challenging that all engineers yearn for. The kind of work that you are trained
to do in university - implementing red-black trees; modifying Dijkstra's
algorithm to suit some arcane need; or coming up with some novel algorithm. The
kind of work that hardly any other job seemed to offer; certainly not my
internships up to that point, which all centered around writing websites in some
form or another.

I got my first interview at Google during my senior year of university and
promptly failed it.

# Interviews and exams

In a traditional company, the programming interview process goes something like
this:

- You write a compelling one-page resume, trying to condense your work
  experience whilst using all buzz words that might be Ctrl-F'ed by a recruiter.
- You apply for a specific job on a specific team.
- Recruiting team Yea/Nay depending on whether you have the right mix of
  buzzwords and years experience.
- If they Yea, you get a phone call talking about the job and the interview
  process.
- Then, you usually do one phone interview with an engineer or manager which
  is designed to vet out the incompetent and clearly poor fits.
- If you make it past this, you're usually invited onsite for something like a
  half or full day. It's usually comprised of you being asked about things on
  your resume, about how you would tackle imaginary problems, and to
  solve contrived problems - or perhaps even solve a real problem. All the while
  you're usually evaluated on some fuzzy criteria of "intelligence" and
  "actually knows wtf they are doing" and "would i want to work with this
  person".

It's a very imperfect process, but it seems to work OK-ish for a lot of
companies.

Google's is much more academic, exam-like. It looks something like this:

- You write a resume that will be scanned for minimum qualifications and then
  largely ignored.
- You apply for a role (ex SWE) with no idea of which team you might be placed in.
- Phone interview with recruiter which lays out the next steps.
- 45m technical phone interview with a randomly-selected SWE. You're writing
  code straight into Google Docs. The interview question is usually hard but
  not mind-blowingly hard. The format is exam-like: you're posed a single
  question and then asked to solve it. If you succeed, the question is altered
  slightly to be harder, and then you're asked to alter your solution to solve
  the new version of the question.
- Next, you may be asked to perform another 45m technical phone interview if
  your last one was borderline.
- Next, you make it to the onsite. Huzzah! The onsite is 5 in-person 45m
  technical interviews at a Google campus. Each of the 45m interviews is once
  again exam-like, and usually a bit harder than the phone interview. You're
  being asked questions you'd expect on a computer science exam: answers usually
  involve things like Minimum Spanning Trees, Heaps, clever bit manipulations,
  and so on. The solution is "coded up" on the whiteboard. Each interviewer is
  selected largely at random from a pool of SWE interviewers.
- Finally, all data from interviews is collated and sent to a hiring committee.
- If the hiring committee Yeas you, you work with the recruiter to pick a team.

Most people take somewhere between 2 weeks to 3 months to study before
attempting the interview (exam).

If you fail anywhere in the process, you must wait one year before being allowed
to repeat the process.

# Failing and failure

My first failure was in 2013 - my senior year of university.

After that failure, I became determined to get better - to _be_ better. I had
a minimum bar for myself and a singular goal I could focus on. One year later in
2014 I re-took the exam and failed again. I did so again in 2015, again in 2016,
and finally in 2017 I made it through. Each year I spent around 2 months
preparing for the interview.

I believe two things changed in ways that helped me get through the interview.

The first is that after failing the SWE interview in 2016 the recruiter
suggested I try interviewing for a Developer Programs Engineer (DPE) role. The DPE role was
described as a more open-source and developers focused role, as opposed to the
only-software-all-the-time focused SWE role. It was still in engineering, and
still involved programming, but the interview was slightly different.

I was all for it, figuring that my several years of open source contribution at
that point might give me an edge, and that if I didn't like it I could probably
switch to a SWE role internally.

So, after accepting the idea, I was immediately fast-tracked into an
onsite DPE interview a few weeks after that failure. I promptly failed the DPE
interview, too. But! The interviewer said "You were close!". So, that was nice.

The interview didn't differ much, except that one or two of the onsite questions
was geared towards open source. A question that I remember, for example, is to
explain in detail how dockerization works from the kernel up to the user facing
pieces. The remaining three or four questions were still the usual solve-a-hard-problem
whiteboard types.

In 2017, I gave the DPE interview another go and succeeded. The open source parts
of the interview were focused on Go - a programming language I had been using
since 2013, and had been active in the community for some time at that point.
It helped that I was very familiar with the subject at that point.

The other part that changed is my method to preparing for interviews.

# Interview preparation

There's about 6 ways you can prepare for a technical interview:

- Memorize algorithms and data structures.
- Solve contrived problems on leetcode.com / hackerrank.com / _Cracking The
  Coding Interview_.
- Read computer science books (such as Skiena's _Algorithm Design Manual_).
- Take a computer science course (or, be a student already taking one).
- Perform real or mock interviews.
- Do hard, Computer-Science-y things regularly (like, as part of your job).

For the majority of my interviews during my career, I had focused on the first
two: memorization and solving contrived problems. In retrospect, these did
little for me except make me more anxious. I had trouble retaining the knowledge
for more than a day or two: without putting any of it into actual practice, it
seemed to just evaporate.

I'd also been taking interviews at companies since graduating - even when
happily employed. I figured the longer you go without interviewing, the more
nerve-racking it becomes, and the more out of practice you become, so why not
interview once or twice a year just to keep in the swing of it. This helped
me be more at-ease during interviews, but ultimately I was lacking mastery of
the subjects.

Note: an ideal situation is that your day-to-day job involves the kind of
hard problems that you'd find in a Google interview. That's a pipe dream for
most developers, though: an overwhelming majority of software development is
pretty banal implementation of web applications, mobile applications, or some
piping-of-data in the backend applications. Loads of engineering, surely, but
not really Computer Science.

In 2017 I started a Computer Science Master's program through Georgia Institute
of Technology (Georgia Tech). It was at the time a novel online course, well
reputed for being extremely rigorous despite being online, and offering an
actual Master of Science degree. (It has since become very popular, and has
instigated a wave of similar programs from other top universities.)

The classes I took were Computer Vision, Building Problem-Solving AIs, and
Computer Networking. Each class had intensive projects that forced me to learn
and re-learn large parts of Math and Computer Science. They were also extremely
well taught, and very, very fun. The AI course, for example, had you build a
game-playing AI that you could pit against other classmates' AIs.

This type of atmosphere allowed for experimentation and competition, and
rewarded both broad use of many Computer Science topics as well as deep dives.
It was for me the perfect way to get back in the academic mindset; to master
Computer Science and math topics; and ultimately to prepare for the interview.

As the interview process began again that year, I supplemented it with basic
familiarity with a wide number of algorithms taken from books, as well as a few
mock interviews.

It seemed to work: I got an offer sometime around October of 2017.

Or, maybe they got sick and tired of me interviewing five years running. :)
There's something to be said for stubborn will!

# Repeating the miracle

If I had to do it all again, or give advice to someone starting the process,
here's what I'd suggest:

First, don't pin all your hopes on Google. The acceptance rate is less than
Harvard by an order of magnitude: the numbers aren't really in your favour.
Once you make peace with that, the rest comes a lot easier.

The good news is that if you do convince yourself to study for Google, you will
_crush_ every other interview you take. Google requires such an obscenely high
level of preparation that it makes other interviews look like child play. So,
apply to Google, but also look for other jobs that would interest you: you will
never be more prepared to interview for them.

Another piece of good news is that the difference between a rejection and a
success is mostly the preparation that goes into the candidate. There is
certainly some luck involved, and being naturally gifted always helps, but
mostly practice - smart practice - is what sets good candidates apart from
the rest.

One more piece of good news is that Google goes to great lengths to make the
process as unbiased as possible. That means, you're not on the hook for trying
to buddy up to the person interviewing you. It also means that the differences
between you and the person interviewing you - be it cultural background, or
viewpoints, or any other differentiating factor - are totally irrelevant. You're
only being evaluated on your objective performance.

So, in conclusion, I think that the real secret sauce is choosing a study
schedule that fits healthily into your life - rather than obsessing about
getting _this one job_ - and choosing a method of study that works for you.
Every person learns differently, and retains knowledge differently. Some of my
colleagues swear by working on practice problems, whilst I strongly prefer the
approach of taking courses and completing homework / projects.

Consider the list of interview preparation techniques above and do some
introspection about how you learn best. And, if you can, try to orient your
job around interesting problems that push you to learn, rather than staying
in your comfort zone.
