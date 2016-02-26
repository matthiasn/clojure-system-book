# Testing

In this chapter, we will discuss testing strategies for both the server and the client parts of the system.

## Unit Tests

## Integration Tests

## Continuous Integration

One important part of building a software system is building the environment that allows us to be confident that it works. For this, continuous integration is essential. There are many options out there. When you want to host the CI environment yourself, I have good experiences with **[Jenkins](https://jenkins-ci.org/)**, which is open source and free. If you use **[Jira](https://www.atlassian.com/software/jira)**, you may want to have a look into **[Bamboo](https://www.atlassian.com/software/bamboo)**, which integrates nicely. I'm just not so sure I like Jira, but that's a different story. 


## Hosted CI

After wasting an incredible amount of precious lifetime on the internally hosted CI environment in my last consulting gig, I thought I should have a look at some hosted options for continuous integration. Luckily, the better ones host open source projects for free, so I set up two with GitHub integration for the systems-toolbox library. I will use both for a while to see which I like better. So far, both seem to do the job, with TravisCI being more generous in supporting multiple open source projects, which suits me particularly well.

**DISCLAIMER**: I don't receive money from either, nor do I know anyone in either companies.


### TravisCI



### CircleCI 
