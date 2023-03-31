## Problem

**Your company built an in-house calendar tool called HiCal. You want to add a feature to see the times in a day when _everyone_ is available.**

To do this, you’ll need to know when _any_ team is having a meeting. In HiCal, a meeting is stored as an instance of a Meeting class with integer properties startTime and endTime. These integers represent the number of 30-minute blocks past 9:00am.

 ```swift
 class Meeting: CustomStringConvertible {

    var startTime: Int
    var endTime: Int

    init(startTime: Int, endTime: Int) {

        // number of 30 min blocks past 9:00 am
        self.startTime = startTime
        self.endTime = endTime
    }

    var description: String {
        return "(\(startTime), \(endTime))"
    }
}
````

For example:

 ```swift
let meeting1 = Meeting(startTime: 2, endTime: 3)  // meeting from 10:00 – 10:30 am
let meeting2 = Meeting(startTime: 6, endTime: 9)  // meeting from 12:00 – 1:30 pm`
````

Write a function mergeRanges() that takes an array of multiple meeting time ranges and returns an array of condensed ranges.

For example, given:
 ```swift
 [
    Meeting(startTime: 0,  endTime: 1),
    Meeting(startTime: 3,  endTime: 5),
    Meeting(startTime: 4,  endTime: 8),
    Meeting(startTime: 10, endTime: 12),
    Meeting(startTime: 9,  endTime: 10)
]
````
your function would return:
 ```swift
[
    Meeting(startTime: 0, endTime: 1),
    Meeting(startTime: 3, endTime: 8),
    Meeting(startTime: 9, endTime: 12)
]
````
**Do not assume the meetings are in order.** The meeting times are coming from multiple teams.

**Write a solution that's efficient even when we can't put a nice upper bound on the numbers representing our time ranges.** Here we've simplified our times down to the number of 30-minute slots past 9:00 am. But we want the function to work even for very large numbers, like Unix timestamps. In any case, the spirit of the challenge is to merge meetings where startTime and endTime don't have an upper bound.

## Breakdown
What if we only had two ranges? Let's take:

  ````swift
  [Meeting(startTime: 1, endTime: 3), Meeting(startTime: 2, endTime: 4)]
  ````

These meetings clearly overlap, so we should merge them to give:

  ````swift
  [Meeting(startTime: 1, endTime: 4)]
  ````
But how did we know that these meetings overlap?

We could tell the meetings overlapped because the _end time_ of the first one was after the _start time_ of the second one! But our ideas of "first" and "second" are important here—this only works after we ensure that we treat the meeting that _starts earlier_ as the "first" one.

How would we formalize this as an algorithm? **Be sure to consider these edge cases:**

1.  The end time of the first meeting and the start time of the second meeting are equal. For example: [Meeting(startTime: 1, endTime: 2), Meeting(startTime: 2, endTime: 3)]
2.  The second meeting ends before the first meeting ends. For example: [Meeting(startTime: 1, endTime: 5), Meeting(startTime: 2, endTime: 3)]

Here's a formal algorithm:

1.  We treat the meeting with earlier start time as "first," and the other as "second."
2.  If the end time of the first meeting is _equal to or greater than_ the start time of the second meeting, we merge the two meetings into one time range. The resulting time range's start time is the first meeting's start, and its end time is _the later of_ the two meetings' end times.
3.  Else, we leave them separate.

So, we could compare _every_ meeting to _every other_ meeting in this way, merging them or leaving them separate.

Comparing _all pairs_ of meetings would take O(n2)O(n2) time. We can do better!

If we're going to beat O(n2)O(n2) time, maybe we're going to get O(n)O(n) time? Is there a way to do this in one pass?

It'd be great if, for each meeting, we could just try to merge it with the _next_ meeting. But that's definitely not sufficient, because the ordering of our meetings is random. There might be a non-next meeting that the current meeting could be merged with.

What if we sorted our array of meetings by start time?

Then any meetings that could be merged would always be adjacent!

So we could sort our meetings, then walk through the sorted array and see if each meeting can be merged with the one after it.

Sorting takes O(nlg⁡n)O(nlgn) time in the worst case. If we can then do the merging in one pass, that's another O(n)O(n) time, for O(nlg⁡n)O(nlgn) overall. That's not as good as O(n)O(n), but it's better than O(n2)O(n2).

## Solution
First, we sort our input array of meetings by start time so any meetings that might need to be merged are now next to each other.

Then we walk through our sorted meetings from left to right. At each step, either:

1.  We _can_ merge the current meeting with the previous one, so we do.
2.  We _can't_ merge the current meeting with the previous one, so we know the previous meeting can't be merged with any future meetings and we throw the current meeting into mergedMeetings.

````swift
  func mergeRanges(in meetings: [Meeting]) -> [Meeting] {

    // make a copy so we don't destroy the input
    var sortedMeetings = meetings.map {
        Meeting(startTime: $0.startTime, endTime: $0.endTime)
    }

    // sort by start time
    sortedMeetings.sort { $0.startTime < $1.startTime }

    // initialize mergedMeetings with the earliest meeting
    var mergedMeetings = [sortedMeetings[0]]

    for i in 1..<sortedMeetings.count {
        let currentMeeting = sortedMeetings[i]
        let lastMergedMeeting = mergedMeetings[mergedMeetings.count - 1]

        if currentMeeting.startTime <= lastMergedMeeting.endTime {

            // if the current meeting overlaps with the last merged meeting, use the
            // later end time of the two
            lastMergedMeeting.endTime = max(lastMergedMeeting.endTime)
        } else {

            // add the current meeting since it doesn't overlap
            mergedMeetings.append(currentMeeting)
        }
    }

    return mergedMeetings
}
````
## Complexity

O(nlg⁡n)O(nlgn) time and O(n)O(n) space.

Even though we only walk through our array of meetings once to merge them, we _sort_ all the meetings first, giving us a runtime of O(nlg⁡n)O(nlgn). It's worth noting that _if_ our input were sorted, we could skip the sort and do this in O(n)O(n) time!

We create a new array of merged meeting times. In the worst case, none of the meetings overlap, giving us an array identical to the input array. Thus we have a worst-case space cost of O(n)O(n).

## Bonus

1.  What if we _did_ have an upper bound on the input values? Could we improve our runtime? Would it cost us memory?
2.  Could we do this "in place" on the input array and save some space? What are the pros and cons of doing this in place?

## What We Learned

This one arguably uses a greedy ↴ approach as well, except this time we had to _sort_ the array first.

How did we figure that out?

We started off trying to solve the problem in one pass, and we noticed that it wouldn't work. We then noticed the _reason_ it wouldn't work: to see if a given meeting can be merged, we have to look at _all_ the other meetings! That's because the order of the meetings is random.

_That's_ what got us thinking: what if the array _were_ sorted? We saw that _then_ a greedy approach would work. We had to spend O(nlg⁡n)O(nlgn) time on sorting the array, but it was better than our initial brute force approach, which cost us O(n2)O(n2) time!