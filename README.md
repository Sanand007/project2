# Project 2 Report

## 1. Collection

### 1.1 What we collected

We collected a wealth of data from GitHub for each team. This data was kept organized in mostly the same way that GitHub organizes it, and consists of the following:

* Issues
 + Issue ID
 + Issue Name
 + Creation Time (time of first event)
 + Action Performed (label/unlabel/milestone/closed, etc)
 + User associated with action
* Milestones
 + Milestone ID
 + Description
 + Creation Time
 + Due Time
 + Closed Time
 + User who created it
* Comments
 + User
 + Issue comment was on
 + Timestamp
 + Text of the comment
* Commits
 + User
 + Timestamp
 + Message

Github organizes their API in much the same way and we saw no reason to break too far from their pattern. In addition, grabbing the raw data and caching it locally allows for fewer potential issues with network connections or API rate limitations, as opposed to using feature detectors that performed analysis directly after retrieving the data and storing the results.

### 1.2 How we collected it

As recommended, we utilized the gitable.py utility to grab the initial sets of data, but later decided that we would like more data than it could provide. We also agreed that inserting the data into a SQL database for query would be a valuable first step. Our current data collection process proceeds as follows:

1. Start up [gitable-sql.py](https://github.com/CSC510-2015-Axitron/project2/blob/master/gitable-sql.py) (our modified gitable.py) with inputs for the target repo and anonymized group name
2. Retrieve all the data from GitHub's API
3. Preprocess it (anonymization and formatting for insertion into database)
4. Create a SQLite database if one does not already exist, and insert all the data into it

Once this is complete, it will spit out a .db file for that repo which can be transferred through email or any other file sharing means and queried at leisure offline.

## 2. Anonymization

Anonymization was performed on both group names and group users, so the individual groups and users will not be identified. Each group in our database was assigned with a number based on the order in which they were imported in the database. To anonymize users within each group, an empty array was created and each time when user ID was detected, the python program searched the array for the ID. If the user ID was not found in the array the ID would be entered into the array. Then, the user ID was replaced with "user" plus the index number of the ID in the array. The code for anonymization is shown as follows.

```
def anonymize(user):
  if anonymizer:
    if user == None : return ''
    idx = user_list.index(user) if user in user_list else -1
    if idx == -1:
      user_list.append(user)
      return "user"+str(len(user_list) - 1)
    else:
      return "user"+str(idx)
  else:
    return user
```

## 3. Tables

All tables in this project 2 paper are located in [Bad Smells Results](https://github.com/CSC510-2015-Axitron/project2#9-bad-smells-results) section, one table per bad smell. Each table reports stink scores summarized from the results of feature detectors corresponding to the particular bad smell. If the numerical result of feature detector for the group is higher than the defined threshold, a stink score "1" is earned.

## 4. Data

The totals listed in the table below include data from all 9 projects, our own included, and comes directly from summing up the number of entries in each SQL table.

| Milestones | Issues | Events | Comments | Commits |
|-----------:|-------:|-------:|---------:|--------:|
|         47 |    578 |   3883 |     1328 |    1760 |

## 5. Data Samples

The following samples are raw tuples taken from our own project's .db file. Where applicable, some attributes have been shortened where the data is too large to comfortably fit on the page.

#### Milestone
id|title|description|created_at|due_at|closed_at|user|identifier
---|---|---|---|---|---|---|---
1|v0.1|(text)|1424538641|1425013200|1425168047|group9/user1|989453
#### Issue
id|name
---|---
1|User Input (moving smily via keyboard)
#### Event
issueID|time|action|label|user|milestone|identifier
---|---|---|---|---|---|---
1|1423616709|closed|closed|group9/user1||233661242
#### Comment
issueID|user|createtime|updatetime|text|identifier
---|---|---|---|---|---
1|group9/user1|1423608247|1423608247|(text)|73800452
#### Commits
id|time|sha|user|message
---|---|---|---|---
1|1428349546|d924a137ebf530fcd56c73980c9fcfbf6de69cdd|group9/user3|(text)

## 6. Feature Detection

We eventually came up with 13 feature detectors utilizing our data. These detectors are as follows:

#### (1) Long Open Issues

This feature is relatively straightforward, we found the difference between the time that an issue was marked as closed and when the first event occurred on that issue. Expressed in SQL, this feature can be detected via:
```SQL
select cl.issueID, (cl.time - op.time) as timeOpen from event cl, (select issueID, min(time) as time from event group by issueID) op where cl.action == 'closed' AND cl.issueID == op.issueID;
```

#### (2) Issues Missing Milestones

This feature is also fairly straightforward, we found the difference between the time that an issue was marked as closed and when the milstone it was assigned to was due. In SQL, this is expressed as:
```SQL
select ev.issueID, ev.time-milestone.due_at as secondsAfter from
(select issueID, time, milestone from event ev1 where action = 'closed' and milestone not null and time >=
   (select max(time) from event where issueID = ev1.issueID and action = 'closed')
) ev,
milestone where milestone.id = ev.milestone;
```

#### (3) Issue Closed Long Before Milstone Due

This feature is a little bit more complicated than the previous ones. In our project we came up with milestones under the assumption that all work for a particular milestone should be complete before a new milestone was started, and the work for that next milestone should not start until the work for the previous milestone had been completed. Operating under this assumption, we decided that it would be improper if work on a milestone's issues (shown as closed issues) was being done before a milestone was even started, meaning the one previous to it had not finished.
This idea can be expressed as an SQL query as the following:
```SQL
select e1.issueID, (e1.time - (mileDur.due_at - mileDur.duration)) as timeAfterStart, mileDur.duration as duration from
event e1,
(select m2.id, m2.due_at, (m2.due_at - m1.due_at) as duration from milestone m1, milestone m2 where m2.id = m1.id+1) mileDur
where e1.milestone = mileDur.id and e1.action = 'closed' and e1.time >= (select max(time) from event where event.action = 'closed' and issueID = e1.issueID);
```

#### (4) Equal Number of Issue Assignees

Rather straightforward, we decided that finding the number of issues assigned to each user in a project would be helpful information. In SQL, this is expressed as:
```SQL
select label, count(*) from (select issueID, label from event e1 where action = 'assigned' and time >= (select max(time) from event where e1.issueID = issueID and action = 'assigned')) group by label;
```

#### (5) Number of People Commenting on an Issue

We felt that communicating on issues was an important part of team cohesiveness and involvement, and we felt that whether a low or high number of peple commented on any particular issue would be an interesting data point to have. In SQL, we can discover the number of unique (non-repeated) users commenting on any particular issue with:
```SQL
select issueID, count(distinct user) from comment group by issueID;
```

#### (6) Number of Issues Posted by Each User

As with users being assigned issues, we felt that knowing the spread of who identified or created issues was important. In SQL:
```SQL
select user, count(*) from event e1 where time <= (select min(time) from event where issueID = e1.issueID) group by user;
```

#### (7) Bug Label Usage

All software has bugs at some point or another, no matter how good the coder or tools, so we felt that the number of bug reports on a project would be a good metric of how carefully the team was looking for them. This detector outputs the number of issues labeled with any label that looks like "bug", the number of total issues, and the ratio of the two.
```SQL
select bugs, count(distinct issueID) as issues, (bugs+0.0)/count(distinct issueID) as ratio from (select count(*) as bugs from event where lower(label) like '%bug%' and action == 'labeled'), event;
```

#### (8) Number of Comments on an Issue

As with the number of people commenting on an issue, we felt the number of comments on an issue was important information.
```SQL
select issueID, count(*) from comment group by issueID;
```

#### (9) Nonlinear Progress Momentum

This feature is difficult to describe succinctly. It is a relation between a milestone's creation date, due date, and actual completion date. Projects that are more waterfall-like will have a very flat line for milestone creation date, showing that milestones were created all at once at the beginning of the project. Very agile-like projects will have a near-constant gap between milestone creation date and due date, indicating milestones are created every so often as work is proceeding. Projects with good effort estimation will have a very small gap between milestone due dates and completion dates, if any, whereas projects with poor effort estimation will have large or highly variable gaps.

Additional feature detection can be performed on milestone due dates. Assuming each milestone in the project requires euqal amount of the effort, the due dates vs. the milestone numbers should yield a linear plot. To quantify the linearity of such curves, the due dates of milestones were fitted by a second degree polynomial regression equation.

```
y = a * x^2 + b * x + c
```

where variable **a** represents the curvature of the curve. High linearity should result in small value (i.e., a ~ 0).


#### (10) Commit History Linearity

The commit history of a project generally indicates how frequently people are working on the project. A completely linear commit history would indicate that there were constant amounts of work occurring over time, but since life is not quite that perfect, a linearity of between 0.0 and 1.0 might be expected for projects that are proceeding smoothly. Graphs with a more expoential appearance, however, may indicate that significant portions of the project are being completed closer to the deadline.

To determine the linearity of commit history, the area under each graph was determined and then compared to the area of the ideal curve. The equation to calcuate the linearity is shown as follows:

![linearity](http://i.imgur.com/zBDoT1n.png)

#### (11) Percentage of Comments by User

As with other participation measures, the percentage of comments in a repo from any particular user can be quite useful, and is expressed in SQL as the following:
```SQL
select user, count(*) as comments from comment group by user;
```

#### (12) Short Lived Issues

We observed in our own group that users would occasionally mentally assign themselves to a feature but forget to create an issue until after the feature was complete. Such behavior may lead to duplicated work or simply be indicative of poor communication. In SQL we can extract the number of issues that were open for less than an hour along with the total number of issues with the following:
```SQL
select numShort, numTotal, (numShort+0.0)/numTotal from (select count(*) as numShort from event cl, (select issueID, min(time) as time from event group by issueID) op where cl.action == 'closed' AND cl.issueID == op.issueID and (cl.time - op.time) < 3600), (select count(distinct issueID) as numTotal from event);
```

#### (13) Effort Estimation Error

This detector was intended to discover if there was a correlation between when a milestone was created relative to when it was due, and how far off it was from the actual close date. If such a correlation existed, milestones created far from their due dates would be more frequently missed, and by a larger margin, than milestones created close to their due dates.

## 7. Feature Detection Results

We created a series of visuals to describe the data we have for each feature detector.  They are each listed below with any notes that are appropriate for them.


#### (1) Long Open Issues

This visualization used boxplots to show the length of time that issues had been open.  It was a calculation of inception of an issue until when it was closed.  This is a bit difficult to categorize as better or worse either way, but issues over 20 days old are not usually the best in our experience, and having a median value at or above that was considered negative.

![long open issues](https://cloud.githubusercontent.com/assets/6590396/7217751/285ea032-e611-11e4-8365-834c1e9a2d9e.png)

#### (2) Issues Missing Milestones

This visualization contextualizes when issues were closed relative to when the milestone they were assigned to was set to be due. The 0 mark is when the milestone for any given issue was due, so bars that overlap into positive values are overdue.

![missing milestones](https://cloud.githubusercontent.com/assets/6590396/7217777/4169ccd6-e612-11e4-8863-1ff80ecba53d.png)

#### (3) Issues Closed Long Before Milstone Due

This visualization shows when issues that were assigned to milestones were closed, it uses the number of days before or after a milestone starts as the scale.  This feature extractor was actually made upon making other visualizations that showed an odd behavior among some of the groups.  As can be seen below there were a number of groups that actually closed a large number of issues before the milestone they were in was actually closed.

![closed before due](https://cloud.githubusercontent.com/assets/6590396/7217786/e2c99016-e612-11e4-86a2-0d4f58a21f50.png)

#### (4) Equal Number of Issue Assignees

A simple pie graph visualization of the precentage of issues that were assigned to a particular user.  Certainly the ideal case would be an even partition among group members. In this and subsequent pie graphs relating to users, group 7's data is inconsequential as it consisted of only one participant.

![equal number of assignements](https://cloud.githubusercontent.com/assets/6590396/7217794/4c070464-e613-11e4-808a-0e1f21d3ab2e.png)



#### (5) Number of People Commenting on an Issue

Again this was created as a pie chart and shows the percentage of comments on an issue on average for each user.  An evenly split graph is best here.

![num comments on issues](https://cloud.githubusercontent.com/assets/6590396/7374415/3a6b0162-eda0-11e4-96e9-0b6d03a45648.png)

#### (6) Number of Issues Posted by Each User

A pie chart that shows the percentage of issues posted by each user.  Similar numbers of issues between each user is better here.

![num issues per user](https://cloud.githubusercontent.com/assets/6590396/7281789/c8fc21a0-e8f9-11e4-87ca-823fe0ae7b8a.png)

#### (7) Bug Label Usage

This turned out to be a somewhat difficult extractor to use, but the vizualization shows the percentage of issues that were non-bug-related and the percentage that were bug- related.  These graphs only apply to groups that actually had bug classification systems.  The ideal case would be for there to be a decent percentage of issues labeled as bugs.  Groups 6, 7, and 8 in did particularly well in this area.  Groups 2, 4, and 5 did not use bug labels, and thus are excluded from this visualization.

![bug label usage](https://cloud.githubusercontent.com/assets/6590396/7311451/86d05780-ea0a-11e4-8302-3290c06b5cc8.png)

#### (8) Number of Comments on an Issue

This is another boxplot used to describe the average number of comments per issue for a particular group.  The idea is that more discussion per issue is better.  As with the participant pie plots, group 7 had only one participant, and that participant did not talk to themselves extensively, hence the very small bar.

![number of comments on an issue](https://cloud.githubusercontent.com/assets/6590396/7281800/d76cadc2-e8f9-11e4-8328-0475e96dd5d8.png)

#### (9) Nonlinear Progress Momentum

This visualization is actually fairly difficult to explain, but the idea was to relate the dates that milestones were planned, due, and closed.  It is somewhat difficult to define the ideal case here, but in some of the groups it is apparent that some of their milestones were never closed.  Similarly, some of the groups closed their milestones well past their due dates.  

The best performance here occurs when the lines, green and blue if not red as well, are similar.  The red line is more of a stylistic issue across the groups.  The first group in the vizualization simply planned all of their milestones at the beginning other groups had a planning line that followed the behavior of their due and closed lines.  Group 5 was excluded from this graph because GitHub stored their milestones differently than the others, and they only had two, thus we felt that for them it was unnecessary to make a regression based on this.

![non linear progress momentum](https://camo.githubusercontent.com/c46f5607d957682cfcb00111f21d52701b4f0a8a/687474703a2f2f692e696d6775722e636f6d2f566d5231555a322e706e67)



#### (10) Commit History Linearity

This visualization describes the rate at which commits were made to a specific group's repository.  The ideal case here would be a straight line with a slope of 1.  That would mean that commits were made each day and at nearly the same rate each day, so the work would have been spread evenly through the semester.  Group 4 was excluded from this visualization because they had only a very small number of commits (2), the linearity regression for them would be untenable.

![commit history linearity](https://camo.githubusercontent.com/79a3f2c59b0b47ddc13fa29747fbb5b20738bd6b/687474703a2f2f692e696d6775722e636f6d2f5979376b4b35732e706e67)

#### (11) Percentage of Comments by User

This boxplot shows the average number of comments per issue, the subheading is percentage of comments by user, but this visualization proved to be more interesting for our work.  It showed that for many groups two users commenting on each issue was pretty good, whereas some groups averaged no more than one user per issue. More members commenting on each issue was considered to be better.  Group 7 only had one member, so all of its issues reflected only having one member commenting on an issue.

![number of users commenting on an issue](https://cloud.githubusercontent.com/assets/6590396/7281781/b7dc18da-e8f9-11e4-838a-3d9010f708ce.png)

#### (12) Short Lived Issues

This pie chart shows the proportion of short issues (open less than an hour) to the total number of issues. A high number of short-open issues implies that issues were not used very effectively to communicate what each member was doing, unless the features the issues represented were in fact small enough to be completed within an hour.

![short open issues](https://cloud.githubusercontent.com/assets/6590396/7448351/732a8a42-f1e5-11e4-9b57-5da4ea9e10af.png)

#### (13) Effort Estimation Error

This scatterplot relates when the milestone was created, relative to when it was set to be due, to when the milestone was actually marked as complete. Scatters that show a trend from upper left to lower right indicate that the group was more accurate at predicting when the due date was closer to when the milestone was created. Scatters that show no trend indicate that there was little relation between prediction accuracy and time until due date.  Group 5 was again omitted because they had only two milestones and GitHub tracked those issues differently than the other groups.

![effort estimation error](https://camo.githubusercontent.com/a494c71daa2a4c8784f5e5a6e1e704788fb8c3ee/687474703a2f2f692e696d6775722e636f6d2f455545553339762e706e67)

## 8. Bad Smells Detector

We combined our feature extractors to create 5 different bad smell detectors. In this section we will attempt to describe how these detectors were created as well as the motivation for each detector.  The idea was to try to create as many detectors as possible from the feature extractors that we were able to define.

#### Poor Communication
The first detector is for poor communication.  The overall motivation with this as a bad smell is that if a group cannot communicate well, their productivity over the long term may be negatively affected.  People in such situations may work discordantly and might run into issues such as commit collisions or effort duplication.  It uses three feature extractors:

- The number of posters in issue comments
- The number of comments in issues
- The percentage of comments by each user

For the number of posters in issue comments, we were looking at the number of of individual users that posted on each issue.  The idea is that if only a small percentage of a group actually discussed issues then there is very poor communication overall.  We set the limit at 50% of of the group, so groups having less than 50% of their members commenting on issues had a bad smell.

Number of comments in issues was gauged a bit differently.  We came to the conclusion that if an issue was important enough to be posted, then there should be some kind of discussion of that issue.  We decided that less than 2 comments per issue was a bad smell.  At that point there is little to no discussion of individual issues.

Percentage of comments by each user is another metric for communication.  If only a few members post comments in issues, then there are only a few members that are actually communicating.  We decided that if an individual user had fewer comments than the output of the following function: (total number of comments/number of group members * 2, then we had a bad smell).

#### Poor Milestone Usage
Milestones can be a great way to set up work for groups by splitting work into achievable sections.  So there can be two major benefits: the first is a simple sense of achievement for a group upon comletion of a milestone, the second is a a deadline to meet to keep work on track.  Additonally milestones can be used to get an idea of whether work is on track or not.  Given all of that poor milestone usage can definitely affect the level of success or failure that a group may find in producing a piece of software.  This detector comibined the following metrics:

- Issues missing milestones
- Nonlinear progress momentum
- Less than 3 milestones overall
- Milestones left incomplete

For the first of the metrics, issues missing milestones, we decided that a median value greater than zero was a poor value for a group. The idea was that having a small amount of issues that were outside of milestones was acceptable because there are bound to be such issues.  A good example is a bug issue post, as it would likely be outside of any milestone, and it is also definitely good to have. 

Non-linear progress momentum is another graphical representation of how groups planned out their project. The ideal curve should be linear, meaning good planning. If nonlinearity factor is more than 3.0 it is considered bad. In addition to the milestone graphs, two additional data points such as closing dates and planning dates also reveal how groups cope with variation during project exeuction. Regardless of linearity of the curves, each milestone should be closed within reasonable timeframe. It is interesting to note that two groups with nonlinear preogress momenum in their milestones have at least one milestone that remains open. 

If there were fewer than three milestones overall, there was also a case for poor milestone usage.  The motivation here is that if a group only had three milestones over the entire project lifespan, then those three milestones were likely packed with issues and would be unweildy.  Splitting among many reachable milestones was viewed as more positive.

Milestones that were not completed was a clear bad smell within a project.  If a group left any milestones incomplete, they were considered as having a bad smell.  Out of the groups that actually left milestones open, both left two milestones open.  In reality this should be viewed even more negatively than one missed milestone, but only one point was given for this.

#### Absent Group Member
The idea with this bad smell was that if a group member was not contributing, they represented sort of dead weight within a project.  There are many bad effects that come with an absent group member, but the most detrimental is poor morale.  If other group members sense that there is a member who isn't pulling their weight, it can negatively affect how the working members feel about the project in general.  Then added to that effect is the simple fact that there are fewer hands to work on the project.  The metrics attached to the smell are:

- Number of issue posters
- Percentage of comments by each user
- Equal number of assignees
- Number of posters in issue comments

We used the same calculation for the first three metrics:  (Total number of x / number of group members * 2) where x is either issues, comments, or assignments for each metric.  If any individual user was below the number calculated for that function in either issues, comments, or assignements as matched the case, we considered that a bad smell.  

Additionally we looked at the number of posters in issue comments.  If a user was posting 50% fewer comments than average per user, we would also consider that a bad smell.

#### Poor Planning
Poor planning was a bit of a hodgepodge of different feature extractors.  The motivation here is that if a group had poor planning there could be many different failings that emerge over a semester.  If a group has planned poorly it would have likely limited the amount of success they could achieve over the life of their project.  The main way that takes place is in the lack of thinking about potential roadblocks in a project, if there is no plan in place then projects could get much of the way through development and hit an issue that makes the entire project non-functional.  Also a group may find that the scale of their project is either way above or way below the amount of time available for the project.  The metrics for the smell are:

- Long time between issues opening and closing
- Issues missing milestones
- Equal number of assignnees
- Non-linear progress momentum
- Short time between issues opening and closing
- Commit history linearity

For long time between issues opening and closing, we considered twenty days a poor performance.  At twenty days an issue had been outstanding for quite a long time and represents a bad smell.

We used the same calculations mentioned above for issues missing milestones, equal number of assignees, and non-linear progress momentum. 

With short time between issues opening and closing, we considered that if greater than 20% of the issues for a group were open less than an hour, that was a bad smell.

Finally with the linearity of commit history, if the error in linearity was greater than 20%, we considered that a bad smell.

#### Dictator 
Our dictator bad smell was very similar to our absent bad smell, but with many of the inequality symbols changed around.  If a project has a dictator there are many negative effects that may not be immediately visible.  The first effect is that options are not discussed so a group may not be taking an optimal path to completion of the project.  Secondly group members may not feel that they can contribute to the project simply because they were not consulted and do not have a grasp of all of the technologies involved.  The metrics for this smell were:

- Number of posters in issue comments
- Number of issue posters
- Equal number of assignees
- Percent of comments by user

For number of posters in issue comments, if a single user had a comment in most of the issues, that was considered a bad smell.  

If a user posted more than the following function of issues that was also a bad smell: (Total number of issues / Number of members * 2)

If a user had more assignments than the following function of assignments that was also a bad smell: (Total number of assignments / Number of members * 2)

If a user posted more than the following function of comments that was also a bad smell: (Total number of comments / Number of members * 2)


## 9. Bad Smells Results

We have used the following acronyms for the various features that we were dealing with:

- numpost:  The number of posters in issue comments
- numcom: The number of comments in issues
- percom: The percentage of comments by each user
- missmile: Issues missing milestones
- nonlin:  Non-linear progress momentum
- fewmile:  Less than 3 milestones overall
- incmile:  Milestones left incomplete
- eqassgn:  Equal number of assignees
- longtime:  Lone time between issues opening and closing
- shorttime:  Short time between issues opening and closing
- lincommitt:  Commit history linearity
- numiss :  Number of issue posters

All of the results are listed below in tables.  We chose to represent each feature in boolean fashion.  If a group had a particular feature that met our guideline for a smell in a particular bad smell decector, that group received a 1 for that feature.  Each bad smell has a calculation for a "Stink Score".  The idea there is that each group should be graded in some fashion to see the extent of the bad smell for that particular group.  Simply having one of the features in each bad smell is definitely negative, but it seems more fair to look at the groups with respect to each other on each bad smell.  The graph below shows an overview of all of the bad smells for each group.

![stinkgraph.png](https://github.com/CSC510-2015-Axitron/project2/blob/master/img/stinkgraph.png?raw=true)

This graph helps a bit to understand the data but is difficult to interpret.  The most important takeaway is that all of the groups had some bad smells, and some bad smells were much worse across the board than others.  For example, poor communication was a problem in all of the groups per this data, but the degree of the problem differed across groups.  Another bad smell that afflicted all of the gropus was absent group member.  Again, this was a question of degree with some gropus actually receiving a point for all four features on that smell.  Poor milestone usage ended up being the lowest scoring bad smell overall with 4 groups not receiving any points at all.  

Another attempted metric here was an overall calculation of bad smells.  We tallied up the stink score across all of the bad smells to create this graph.  The important thing here was to remove any duplicate values from the calculation.  Eqassign was a duplicate from poor planning in absent group member, and since the split value in numpost was 50% the values were equivalent in both absent group member and dictator group member.  After removing those duplications we ended up with the graph below to document the "stink score" for each group overall.

![totalsting.png](https://github.com/CSC510-2015-Axitron/project2/blob/master/img/totalstink.png?raw=true)


It is very difficult to make any sweeping generalizations about this data given the fact that it was anonymized prior to our analysis.  That the data was anonymized makes it difficult to test whether these metrics actually showed poor performance or not.  That said, with the data as it is we would assume that groups 2, 8, and 1 would have had more difficulty in producing what they were able to produce over the semester.  


#### Poor Communication

The table below describes the scoring for each group in the three categories for poor communication:

| Group Number | numpost | numcom | percom | Stink Score |
|:------------:|:-------:|:------:|:------:|:-----------:|
|       1      |    1    |   1    |   1    |      3      |
|       2      |    1    |   1    |   1    |      3      |
|       3      |    1    |   0    |   0    |      1      |
|       4      |    1    |   1    |   1    |      3      |
|       5      |    0    |   0    |   1    |      1      |
|       6      |    1    |   1    |   1    |      3      |
|       7      |    0    |   1    |   0    |      1      |
|       8      |    1    |   0    |   0    |      1      |
|       9      |    1    |   0    |   1    |      2      |


#### Poor Milstone Usage

| Group Number | missmile | nonlin | fewmile | incmile | StinkScore |
|:------------:|:--------:|:------:|:-------:|:-------:|:----------:|
|       1      |    1     |    1   |    0     |     0    |      2     |
|       2      |    1     |    0   |    0     |     0    |      1     |
|       3      |    0     |    0   |    0     |     0    |      0     |
|       4      |    0     |    0   |    0     |     0    |      0     |
|       5      |    0     |    0   |    0     |     0    |      0     |
|       6      |    0     |    0   |    0     |     0    |      0     |
|       7      |    1     |    1   |    0     |     1    |      3     |
|       8      |    1     |    1   |    0     |     1    |      3     |
|       9      |    0     |    0   |    0     |     0    |      0     |

#### Poor Planning

| Group Number | longtime | missmile | eqassgn | nonlin | shorttime | lincommit | Stink Score |
|:------------:|:--------:|:--------:|:-------:|:------:|:---------:|:---------:|:-----------:|
|       1      |    0     |    1     |    0    |   1    |     0     |     0     |      2      |
|       2      |    1     |    1     |    1    |   0    |     0     |     1     |      4      |
|       3      |    0     |    0     |    0    |   0    |     1     |     0     |      1      |
|       4      |    0     |    0     |    1    |   0    |     0     |     0     |      1      |
|       5      |    1     |    0     |    0    |   0    |     0     |     1     |      2      |
|       6      |    0     |    0     |    1    |   0    |     0     |     1     |      2      |
|       7      |    0     |    1     |    0    |   1    |     0     |     0     |      2      |
|       8      |    0     |    1     |    1    |   1    |     1     |     0     |      4      |
|       9      |    0     |    0     |    0    |   0    |     0     |     0     |      0      |

#### Absent Group Member

| Group Number | numiss | percom | eqassgn | numpost | StinkScore |
|:------------:|:------:|:------:|:-------:|:-------:|:----------:|
|       1      |   1    |   1    |    0    |    1    |     3      |
|       2      |   1    |   1    |    1    |    1    |     4      |
|       3      |   1    |   0    |    0    |    1    |     2      |
|       4      |   1    |   1    |    1    |    1    |     4      |
|       5      |   1    |   0    |    0    |    0    |     1      |
|       6      |   1    |   1    |    1    |    1    |     4      |
|       7      |   0    |   1    |    0    |    0    |     1      |
|       8      |   1    |   0    |    1    |    1    |     3      |
|       9      |   0    |   0    |    0    |    1    |     1      |

#### Dictator Group Member

| Group Number | numiss | percom | eqassgn | numpost | StinkScore |
|:------------:|:------:|:------:|:-------:|:-------:|:----------:|
|       1      |   0    |   1    |    0    |    1    |     2      |
|       2      |   1    |   0    |    0    |    1    |     2      |
|       3      |   1    |   0    |    0    |    1    |     2      |
|       4      |   0    |   0    |    1    |    1    |     2      |
|       5      |   0    |   0    |    0    |    0    |     0      |
|       6      |   0    |   0    |    1    |    1    |     2      |
|       7      |   0    |   0    |    0    |    0    |     0      |
|       8      |   1    |   0    |    0    |    1    |     2      |
|       9      |   0    |   0    |    0    |    1    |     1      |

## 10. Early Warning

We examined all 13 feature detectors and picked No.9 (Nonlinear Progress Momentum) as our early warning detector. This detector only looks at the due dates of milestones within each group. Our justification is as follows:

- Most of the groups had planned their milestones about 40 days before the project was due.
- All groups except one have proposed at least 4 milestones.
- Milestones serve as the checkpoints for measuring the progress of the group.
- Each milestones within the group requires equal amount of effort and time to work on.

Based on the above assumption, the linearity of milestone due dates vs. milestone number should reveal the planning strategy of the group. The regression equation for calculating the linearity is as decscribed previously in this paper. The linearity was measured by the parameter of the second degree polynomial. That is to say, high linearity resulted in very small values.

## 11. Early Warning Results

To evaluate the effectiveness of our early warning detector, the total stink score and the exact nonlinearity factor for each group are listed in the following table.

| Group Number | Total StinkScore | nonlin factor |
|:------------:|:----------------:|:-------------:|
|       1      |       12         |      3.21     |
|       2      |       14         |      2.71     |
|       3      |        6         |      0.13     |
|       4      |       10         |      0.25     |
|       5      |        4         |       N/A     |
|       6      |       11         |      1.06     |
|       7      |        7         |      5.74     |
|       8      |       13         |      6.99     |
|       9      |        4         |      0.01     |

We found that the top three groups having the lowest stink score also have low nonlinerity factor. Respectively, they are 0.01, N/A, and 0.13. Here, the nonlinearity factor for group 5 was not determined because there are only two milestones defined by the group. However, the group that had a low nonlinearity factor does not necessarily have low stink score, which means that good planning does not guarantee better execution. If we examine the top three groups having the highest stink scores (14, 13, and 12) all of them have high nonlinearity in their milestones (2.71, 6.99, and 3.21). This finding agrees with the belief that poor project initiation and unrealistic expections is one of the causes for project failure.


---

# Project 2 Info (Not part of report)
repo for storing project 2 data and code

## Usage
To acquire data in easier to process form, you must run gitable-sql.py
But, to do so, you must have a token.

`curl -i -u <your_username> -d '{"scopes": ["repo", "user"], "note": "OpenSciences"}' https://api.github.com/authorizations`

The token returned in the response should be copied and pasted into gitable.conf (copy gitable.conf.example) in the indicated place.

Once the token is installed, you can run the data dumper via python:
```
python gitable-sql.py repo group1
```
With whatever repo you like and an anonymizing group name. After a few minutes of churning over web requests, it'll spit out a sqlite3 database file with the data packed up and ready for query.

## Known limitations
Currently, if one of the many requests made to GitHub for data times out, the entire run is pretty much useless. In the future we hope to capture the error as it is happening and have the data dumper retry timed out queries.
