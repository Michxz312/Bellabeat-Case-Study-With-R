# Bellabeat-Case-Study-With-R

# Executive Summary
This analysis explores Fitbit usage data to identify trends in wellness tracking behaviors, with the goal of informing Bellabeat’s marketing strategy for its Leaf wearable.

Automatic tracking features (like step counting) drive higher engagement, while manual features (like weight logging) are underused. Users who consistently log sleep data tend to sleep longer, suggesting a positive feedback loop between tracking and behavior change. A strong correlation between steps and calories burned highlights how physical activity drives visible health outcomes, making it possible to optimize performance. 

Based on these findings, I developed data-informed marketing strategies focused on highlighting Leaf's automatic stress and reproductive health tracking, targeting wellness-focused women with habit-building and consistency, promoting user stories that reflect behavior change over time.

Tools: R, tidyverse, and ggplot2 
This project was completed as part of the Google Data Analytics Certificate.

# Introduction

### BellaBeat
A woman's wellness company that has helped millions of women track their cycle, pregnancies, and live more in sync with their cycles. [https://bellabeat.com/about-us/](http://)

**Business Goals:**
Empower women to take control of their health by providing them with technology-driven designs that blend design and function. 

**Products:**
* Bellabeat app: provides users with health data related to their activity, sleep, stress, menstrual cycle, and mindfulness habits
* Leaf: connects to the Bellabeat app to track activity, sleep, and stress
* Time: track the user activity, sleep, and stress
* Spring: water bottle that tracks daily water intake
* Bellabeat Membership: 24/7 guidance on nutrition, activity, sleep, health and beauty, and mindfulness based on their lifestyle and goals

**Advertising method:** digital marketing (Google Search, Facebook pages, Instagram, Twitter engagement)

# Problems

## Business Task

Focus on a product and analyze smart device usage data in order to gain insights into how people are already using their smart devices. Find high-level recommendations for how these trends can inform Bellabeat marketing strategy.

## Goal
The Bellabeat Leaf tracks activity, sleep, stress, meditation, and reproductive health. Since the Fitbit data focuses primarily on activity and sleep, analyzing user behavior in these areas allow us to identify trends and create marketing strategies for Leaf, especially for users already engaged with similar tracking habits.

1. What are some trends in smart device usage?
2. How could these trends apply to Bellabeat customers?
3. How could these trends help influence Bellabeat marketing strategy?

# Load Fitbit datasets

```{r setup}
knitr::opts_chunk$set(echo = TRUE)
library(tidyverse)
library(readxl)
library(sugrrants)
library(janitor)

daily_activity <- read_excel("Fitbit Data/dailyActivity_merged.xlsx")
sleep_log <- read_excel("Fitbit Data/sleepDay_merged.xlsx")
weight_log <- read_excel("Fitbit Data/weightLogInfo_merged.xlsx")

```

## Exploratory Data Overview

### Checking and Removing Duplicates
```{r}
daily_activity <- daily_activity %>% distinct()
sleep_log <- sleep_log %>% distinct()
weight_log <- weight_log %>% distinct()
```

### Cleaning Column Names
To improve readability and consistency, we use the clean_names(data, case='snake') function to change all column names into lower case and replace spaces or uppercase with an underscore.
```{r}
daily_activity <- clean_names(daily_activity, case='snake')
sleep_log <- clean_names(sleep_log, case='snake')
weight_log <- clean_names(weight_log, case='snake')
colnames(daily_activity)
colnames(sleep_log)
colnames(weight_log)
```

After cleaning the names, want to understand how many unique users are representerd in each dataset. Since each user can have multiple entries (across different dates), it's important to count distinct user IDs.
```{r}
n_distinct(daily_activity$id)
n_distinct(sleep_log$id)
n_distinct(weight_log$id)

```
There are significantly fewer users who have logged their weight compared to those who have logged activity or sleep data. When Fitbit is worn to bed, it automatically tracks the users' sleep, but weight needs the users' manual input.

## Transforming the Data set

The date columns in the dataset include both date and time, but for the following analysis, we only need the date in the format of 'mm/dd/yy'. Therefore, we will convert these columns into the proper date using as.Date() function.
```{r}
weight_log <- weight_log %>%
  mutate(date = as.Date(date, format = "%m/%d/%Y"))

sleep_log <- sleep_log %>%
  mutate(date = as.Date(sleep_day, format = "%m/%d/%Y"))

daily_activity <- daily_activity %>%
  mutate(date = as.Date(activity_date, format = "%m/%d/%Y"))
```


## Summary of each dataframe

For each of the data frame, we use the summary() function to generate basic descriptive statistic for the key variables in the dataset. This provides a quick overview of users' physical activity patterns.
### Daily Activity
```{r}
daily_activity %>% summary()
```
The summary statistics provide an overview of user behavior in terms of physical activity and calories burned:
* id: Represents unique users in the dataset
* activity_date/date: Indicates when each activity was recorded. Useful for analyzing trends over time.
* total_steps and total_distance: Both of these variables can be used interchangeably because they reflect the user's overall daily movement.
* calories: Ranges from 0 to 4,900 with a mean of ~2,304. This metric shows the total calories burned and can be used to assess the relationship between physical activity and energy expenditure.
* active_minutes (very_active_minutes, fairly_active_minutes, lightly_active_minutes, sedentary_minutes): These variables represent the duration of time users spend at different intensity levels. The variables are important for understanding how different activity types contribute to calories burned and for segmenting users by activity patterns.
* Redundant or less informative variables:
  * active_distance: Relatively low values across users, suggesting minimal variation or usage. This metric may therefore offer limited analytical value compared to active minutes.
  * tracker_distance: Nearly identical to total_distance.
  * Logged_activities_distance: Mostly zero entries, which suggests users rarely manually log distance. 

### Sleep Activity
```{r}
sleep_log %>%
  select(total_minutes_asleep, total_time_in_bed) %>%
  summary()
```
The sleep_log dataset provides insights into users' sleep behavior. For this analysis, the focus is on the two key variables:
* total_minutes_asleep: The average is about 419.5 minutes which is 6.99 hours of sleep, which aligns closely with the recommended amount of sleep for adults. We also see that the minimum sleep recorded is 58 minutes, suggesting that nap times may be included.
* total_time_in_bed: This variable reflects the overall time users spent in bed and is closely related to total_minutes_asleep. By comparing these two values, we can estimate sleep efficiency, which is useful for identifying sleep quality (time asleep vs time in bed).

### Weight Log
```{r}
weight_log %>%
  select(weight_kg, bmi) %>%
  summary()
```

Although only 8 distinct users log weight-related data, analyzing weight_kg and bmi can provide useful indicators of progress or behavioral change among the small group of engaged users.
* weight_kg: Records users' body weight at different points in time.
* bmi: Provides context for weight by accounting for user height.

**Note: There are limited samples for this analysis.**

# Merging Datasets
In order to get all users who log all three types of data, only activity and sleep, only activity, only weight, etc..., a full_join() function should be used to preserve all rows. This merge will create missing values which identifies what user didn't log. 
```{r}
# Get the list of all users with all three logs
merge_all <- daily_activity %>%
  full_join(sleep_log, by=c("id", "date")) %>%
  full_join(weight_log, by=c("id", "date"))

```

```{r, include = FALSE}
# Select only relevant columns
merge_all <- merge_all %>% 
  select (
    id, date, total_steps, very_active_minutes, fairly_active_minutes, lightly_active_minutes, sedentary_minutes, calories, total_minutes_asleep, total_time_in_bed, weight_kg, bmi)
```

From the merge_all dataframe, we can create another dataframe that tells us how many days each user logged sleep, weight, and activity.

```{r}
tracking_stats <- merge_all %>%
  group_by(id) %>%
  summarise(
    days_with_sleep=sum(!is.na(total_minutes_asleep)),
    days_with_activity=sum(!is.na(total_steps)),
    days_with_weight=sum(!is.na(weight_kg))
  ) %>%
  select(
    id, days_with_sleep, days_with_activity, days_with_weight
  )

min(tracking_stats$days_with_activity)
```

The minimum number of days that a user log their physical activity is 4 days. This shows that all users log their physical activity whether it's manually or automatically. Next, we will figure out exactly how many users log the activity types. The following dataframe shows the category of users by logging pattern.

```{r, echo=FALSE}
user_segments <- tracking_stats %>% 
  mutate(
    segment = case_when(
      days_with_activity > 0 & days_with_sleep > 0 & days_with_weight > 0 ~ "All Three",
      days_with_activity > 0 & days_with_sleep > 0 & days_with_weight == 0 ~ "Activity + Sleep",
      days_with_activity > 0 & days_with_sleep == 0 & days_with_weight > 0 ~ "Activity + Weight",
      days_with_activity > 0 & days_with_sleep == 0 & days_with_weight == 0 ~ "Activity Only",
      TRUE ~ "Other"
    )
  )

# show count of how many users log each type
user_count <- user_segments %>% count(segment)
user_count
```

# Analysis

### Average Steps vs Calories Burned
```{R}

# subset for average steps per day and calories by user
avg_steps_calories <- daily_activity %>%
  group_by(id) %>%
  summarise(
    avg_steps = mean(total_steps, na.rm = TRUE),
    avg_calories = mean(calories, na.rm = TRUE)
  )

# plot for calories
ggplot(avg_steps_calories, aes(x= avg_steps, y=avg_calories)) + 
  geom_point() + geom_smooth(method = "lm", se = FALSE, color = "blue", linetype = "dashed")
```

The plot shows a positive correlation with the average calories and average steps of each user. As expected, there is a positive correlation between steps and calories burned. However, the wide spread of points around the line suggests that other activities (i.e. high-intensity workouts) may burn more calories than step count alone would indicate. This highlights the value of smart wearables like the Bellabeat Leaf, which go beyond simple step tracking. By providing users with a fuller picture of their physical activity, these devices can help optimize health outcomes and encourage more personalized wellness journeys.

<!--Plot log frequency per user-->
```{R, echo=FALSE}
# Reshaping the tracking_stats to make histogram
tracking_stats_long <- tracking_stats %>%
  pivot_longer(
    cols=c(days_with_sleep, days_with_activity, days_with_weight),
    names_to='log_type',
    values_to='days_logged'
  )

ggplot(tracking_stats_long, aes(x=log_type, y =days_logged, fill=log_type)) +
  geom_boxplot() +
  labs(
    title = "Distribution of Logging Days per Log Type",
    x = "Log Type",
    y = "Days Logged"
  ) +
  theme_minimal()

```

Users log their activity the most, their sleep less frequently, and their weight the least. There is high variability in how frequently users log their sleep.

Among 33 users, activity tracking is the most consistent, likely due to automation. Sleep tracking is used by some, but most users don't log it consistently with the mean being 5 days. Weight tracking is rare, likely due to manual input. 

### A closer look at a consistent user

<!--Heat map calendar for Id = 6962181067-->
After a quick look at the tracking_stats subset, we find that there is a user who log their weight, sleep, and activity for about 30 days, which is the maximum days that a user log their weight.
```{r, echo=FALSE}
# Get id of consistent track
consistent_user <- tracking_stats %>% filter(
  days_with_weight==max(days_with_weight, na.rm=TRUE) 
) %>% pull(id)

user_activity <- merge_all %>% 
  filter(id==consistent_user)

user_activity <- user_activity %>%
  mutate(
    weekday=wday(date, label=TRUE, week_start=1),
    week= isoweek(date),
    date_label = format(date, "%b %d"),
    year=year(date)
  )

# Plot calendar against Total Steps
ggplot(user_activity, aes(x=weekday, y=-week, fill=total_steps)) + 
  geom_tile(color='white') + 
  geom_text(aes(label = date_label), size=3, color="black") +
  scale_fill_gradient(low = "white", high = "red", na.value = "grey95") + scale_x_discrete(labels = c("Mon", "Tue", "Wed", "Thu", "Fri", "Sat", "Sun")) +
  labs(title = "Steps per Day (April 12 to May 12)") +
  theme_void() + 
  theme(
    axis.text.x = element_text()
  )

```
### Calories Heatmap Calendar

```{r, echo=FALSE}
# Plot calendar
ggplot(user_activity, aes(x=weekday, y=-week, fill=calories)) + 
  geom_tile(color='white') + 
  geom_text(aes(label = date_label), size=3, color="black") +
  scale_fill_gradient(low = "white", high = "steelblue", na.value = "grey95") + scale_x_discrete(labels = c("Mon", "Tue", "Wed", "Thu", "Fri", "Sat", "Sun")) +
  labs(title = "Calories per Day (April 12 to May 12)") +
  theme_void() + 
  theme(
    axis.text.x = element_text()
  )

```

### TotalSleep
```{r, echo=FALSE}
# Plot calendar
ggplot(user_activity, aes(x=weekday, y=-week, fill=total_minutes_asleep)) + 
  geom_tile(color='white') + 
  geom_text(aes(label = date_label), size=3, color="black") +
  scale_fill_gradient(low = "white", high = "green", na.value = "grey95") + scale_x_discrete(labels = c("Mon", "Tue", "Wed", "Thu", "Fri", "Sat", "Sun")) +
  labs(title = "Total Minutes Asleep per Day (April 12 to May 12)") +
  theme_void() + 
  theme(
    axis.text.x = element_text()
  )

```

These calendar heatmaps visually show how user behavior changse over time - across steps, calories, abd sleep. The color intensity makes it easier to spot patterns in consistency and effort over the 30 days.

The user's step count was most consistent during the third week, followed by a slight decline. However, overall activity increased over the course of the month, suggesting growing engagement with wellness tracking. Calories per day appear more consistent during the last two weeks compared to the first two weeks, suggesting a more stable calorie burn. The total minutes asleep per day alone doesn't reveal much, but comparing the first and last week shows the user is getting more sleep by the end of the period.

Overall, this suggests the user is engaging with their wellness more over time and there's evidence of increased physical activity, more consistent calorie burn, and improved sleep, showing how someone becomes more mindful of their health.

### Creating an engagement score 
<!--Plot for engagement vs sleep time-->
```{r, echo=FALSE}
# Create a normalized score for sleep and activity
tracking_stats <- tracking_stats %>% 
  mutate(
    normalized_sleep_log=(days_with_sleep - min(days_with_sleep, na.rm=TRUE))/(max(days_with_sleep, na.rm=TRUE)-min(days_with_sleep, na.rm=TRUE))
  )

normalized_sleep_data <- sleep_log %>%
  group_by(id) %>% 
  summarise(total_minutes_asleep = mean(total_minutes_asleep, na.rm=TRUE)) %>%
  inner_join(tracking_stats, by='id')

ggplot(normalized_sleep_data, aes(x = normalized_sleep_log, y = total_minutes_asleep)) +
  geom_point(color = "steelblue", size = 3, alpha = 0.7) +
  geom_smooth(method = "lm", se = FALSE, color = "darkred", linetype = "dashed") +
  labs(
    title = "Sleep Log Frequency vs Average Sleep Duration",
    x = "Normalized Sleep Logging Frequency",
    y = "Average Sleep per User"
  ) +
  theme_minimal()

```

This positive correlation suggests that users who regularly log their sleep tend to sleep more consistently or longer. This supports the idea that self-tracking encourages mindful behavior, a key value proposition for Bellabeat Leaf, which promotes self-awareness around wellness.

### Segmenation by Users' Logging Patterns
Not all users engage with fitbit the same way. To better understand the types of users, we can segment them into groups based on the types of data they track: all three(activity, sleep, and weight), activity and sleep only, or activity only. 
```{r, echo=FALSE}
user_segments <- user_segments %>% select(id, segment)
merge_all <- merge_all %>% left_join(user_segments, by='id')
# ensure that we don't plot it against a null value
# average the total steps for all user
avg_steps_by_segment <- merge_all %>%
  group_by(id, segment) %>%
  summarise(
    avg_steps = mean(total_steps, na.rm=TRUE)
  )

ggplot(avg_steps_by_segment, aes(x = segment, y = avg_steps, fill = segment)) +
  geom_boxplot() +
  labs(title = "Average Steps by User Segment", x= "Type", y = "Average Steps") +
  theme_minimal()
```

Users who log all three data types (activity, sleep, and weight) appear less active than expected. However, this segment includes a very small number of users and is skewed by an outlier with a daily step count of zero. More data is needed to draw reliable conclusions for this group.

Users who track only physical activity have the lowest average step counts, which may indicate more casual engagement with their wearable. These users are likely passive users who aren't leveraging the full functionality of the device.

In contrast, users logging activity and weight demonstrate the highest step averages (~12,000 per day), suggesting a more goal-oriented or health-conscious behavior, possibly motivated by weight management. However, this group also has a small sample size (n = 2), so additional user data would be necessary to validate this trend.

Users who track activity and sleep show moderate step counts, with a wide interquartile range. This variability suggests a mix of user types, but overall, this group may represent users interested in general wellness rather than high-performance fitness goals.

# Conclusion

Activity tracking is the most consistent and automated form of engagement, while sleep tracking is less frequent and weight tracking is rare, likely due to manual effort required. Users who actively log multiple health metrics over time demonstrate positive behavioral trends, including more consistent calorie expenditure and improved sleep. 

There is a high correlation between step count and calories burned, which can highlight how some activities (high-intensity workout) may burn more calories. For users who are interested in optimizing their performance or gains, promoting Leaf's ability to track physical activity levels creates a connection with potential customers.

These findings indicate that Bellabeat's Leaf is most effective when it is simplified and automates data tracking, making it easier for users to maintain consistent engagement. By uncovering these trends, Bellabeat's marketing strategy should emphasize convenience, particularly highlighting automated tracking. Furthermore, with Bellabeat's Leaf tracks activity, sleep, stress, meditation and reproductive health automatically by syncing with the app. 

Through supporting and encouraging consistent, comprehensive health tracking, Bellabeat helps women make informed decisions about their well being. The following high-level recommendations outlines how Bellabeat can use marketing strategies to boost users' engagement and introduce new users to Bellabeat.

# Recommendations
1. Platform: Google Search
Objective: Capture users actively searching for health devices for tracking their physical health
User Behavior: Users comparing features between brands
Target: Health-conscious women comparing wearables
Tactics: create comparison-focused content highlighting unique features like reproductive health tracking and stress management

2.  Platform: Facebook and Instagram
Objective: Drive awareness on the product and find emotional connection through storytelling
User Behavior: Browsing casually
Target: busy professionals, health-conscious users without optimizing performance
Tactics: Share successful stories from Bellabeat users' on how they become more mindful of their health, and how they optimize their calorie burn through Bellabeat's product features

3. Platform: Twitter
Purpose: Engage in conversations
User Behavior: Looking for health topics
Target: well-informed users following health brands
Recommendation: Encourage users in logging their menstrual cycle and Bellabeat can predict future cycles, highlighting automatic tracking
