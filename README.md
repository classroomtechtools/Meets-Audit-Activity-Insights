# Meets Audit Activity Insights

Outputs leadership boards of how many hours per day users are spending in Google Meets, and how frenquently they had meets. 

![FrequencyLeaderboard](/frequencyleaderboard.png?raw=true "Frequency Leaderboard")

It's a spreadsheet that contains all meet data over the last X days. The built-in script interacts with the `Activities.list` [endpoint](https://developers.google.com/admin-sdk/reports/v1/reference/activities/list) available on GSuite domains; it's the same endpoint used in its own reporting console, such as the Meets Quality Tool.

It downloads row-by-row all of this raw information, going back `days` number of days, saves into the spreadsheet into `meet data` sheet.

Additional sheets contain pivot tables that provide insights into the data, such as duration of calls, frequency of use. Leadership boards are derived from those pivot tables.

## License

MIT, credit [@clssrmtechtools](https://twitter.com/clssroomtechtools) Adam Morris [Website](https://classroomtechtools.com), [Email](classroomtechtools.ctt@gmail.com)

## Intended Audience

Leaders in school in distance learning mode who want to see how their Google Meets service is being used, curious GSuite admins, data enthusiasts, AppsScripts writers of libraries, and spreadsheet junkies.

## Requirements

The user will need to permission to access Admin Reports on the domain.

## Quickstart

Since this is an OpenSource tool for fellow GSuite admins, I see no real downside to releasing the code in the following manner: It's a spreadsheet with script contained. You're supposed to open up the Script Editor and execute it.

1. Make a [copy of this spreadsheet](https://docs.google.com/spreadsheets/d/1wOrv2KxLxJwB27OL1butFn_PJPNJlG6VUXpR6bSe0kA/copy);

2. Change the number of days you'd like to download data for. A domain with a large number of users can take a while. On a domain with 300 users with 21 days of data, it took 10 minutes to complete execution (and update values based on formulas);

3. Tools —> Script Editor;
 
4. Execute, and wait.

5. Update the ranges in the pivot tables in tabs `Duration` and `Frequency` so that it captures all of the rows of data. 

6. Yes, you could make this an add-on, which I welcome. All of this it MIT licensed opensource, baby.


## Motivation

Schools are places which need to have safeguarding techniques in place, and one of them is to be able to see activity happening at a glance. This can help answer questions such as "how long are kids spending on video calls?"

Thanks to [@schoraria911](https://twitter.com/schoraria911), who provided some additional motivation in our interactions on twitter.

## Problem solving

### Pivot tables are wonderful, with caveats

I was able to aggregate the raw data with pivot tables, which allowed me to derive frequency tables. While very convenient, the ready-made pivot table feature of google sheets has the following issues:

* When a filter is enabled, it doesn't automatically populate when new data is written
* With the above, there is extraneous information which affects the presentation
* Sorting to get a leadershipboard is a nuisance

The solution for the above issues was to just create a new tab that extracted the information and dissected it with a few good formula, as needed. In the case of determining frequency of use per day, I made a "step" sheet that aggregated the unique days and frequency, converted into hours and the did the math to determine how many hours per day, on average, the particular user is on a call.

### Working with jsons

The raw data from the API endpoint comes back in jsons, which doesn't fit well into a spreadsheet. So what I needed was a piece of code that could take a bunch of jsons and convert them into "rows" or 2d arrays. This is an interesting problem that I had solved previously from another project.

The key is to take nested jsons and convert them into flat jsons, where each key uses dot-and-brackets notation to write the original path to the value. Easy example:

```js
let obj = {
  an: {
      array: [1, 2]
  },
  path: {
     to: {
        value: 'value'
     }
  }  
};

// convert that to:
obj = {
   'an.array[0]': 1,
   'an.array[1]': 2,
   'path.to.value': 'value'
};
```

If all my jsons can be converted to that, then we have our headers for our columns, and rows would just be the values. Easy!

The javascript library [dot-object](https://www.npmjs.com/package/dot-object) was available, which provided an easy way to work with javascript objects in the manner described above, turning a nested objects into a single flat object.

So I wrote a library [dottie](https://github.com/classroomtechtools/dottie.gs) that brought dot-object's ability to AppsScripts. I could then add a method `jsonsToRows` which converted an array of jsons into spreadsheet-friendly rows. In that way, all I would have to do is call `.setValues` and we have all the data written to a spreadsheet.

Simple right? If only!

Just like in any project, there was an unexpected corner case which needed a solution.

### Parameter objects

So I had all the Meets data written to rows, and when I started looking at making pivot tables, I realized that the columns didn't match up.

```
email   |  events[0].parameters[0].name | events[0].parameters.value
a@b.com |  ip_location                  | MY
```
but then the next row would have this:
```
b@c.com | duration                      | 12345
```

That is how the data itself was organized, where not all of the rows had the the datapoints matching in the columns. Had I made a huge blunder somewhere? Checking out the documenation of the response at this endpoint, this is what it gives as the format of the parameters field:

```js
  "parameters": [
            {
              "name": string,
              "value": string,
              "intValue": long,
              "boolValue": boolean,
              // more...
            }
   ]
```

Ugh. So instead of the name being a property name, it was a value which appeared in different order in the data.

So I had to code up a loop in there that updated the event object itself with properties defined by `name` field, whose value was either `value` or `intValue` or `boolValue`.

(If you're scratching your head wondering why on earth would Google do it this way, it's because this particular endpoint has to have parameter names as values, since it's an endpoint shared throughout the entire reporting ecosystem. The Meets features has things like `ip_location` and `duration` but the email reporting tools will have different values for parameters.)

### Advanced Service not advanced enough

The project needs a way to reach out to an API endpoint and returns the data in json format. The AppsScripts platform already has "advanced service" for this many endpoints, including `AdminReports`, but when I first tried it, it took an endless amount of time to work. It was downloading all the information for all users. Even when I downloaded all the information for just one user, it caused me significant delay while I was building it.

Instead, I took a lower-level approach, and interacted with the endpoint directly instead of using `AdminReports`. I had already written a library that abstracted away some of the deails of this, and within a few minutes I was able to get 10 or so data points. No need for me to wait minutes just to get to the next stage of the project. 
