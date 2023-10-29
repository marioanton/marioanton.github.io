---
layout: post
title: Dynamic alerting with alertmanager
image: /img/am.png
tags: [alertmanager,routing,dynamic]
---

Last week i decided to improve the **readability** and reduce the config file **size** of my **alertmanager** **configuration**. Few different receivers for several teams, loads of duplicated config, so better **DRY**.

After banging my head a little bit thinking of wrapping template creation etc which would add a bit of complexity in the process and maintenance overhead, i decided to look at alertmanager precious, but sometimes vague, documentation.

### Finders keepers
Turned to be that some bits of the receivers can be templated! (using nasty gotpl but still has a point)
That meant that i could do, using data coming from some place? (alerts?) send the alert to a specific channel based of dynamic data, use one receiver to rule them all (or as many as you need eh)

My issue resided mostly in **slack** so here go.
Hence i found this in the **alertmanager** config
Congratulations buddy!!
```
# The channel or user to send notifications to.
channel: <tmpl_string>
```

So, i thought, lets try it out.
### What do i need to start using this?
---
 **Data Normalization first.**
I needed to have some data normalisation across the board, which sometimes is not the case but i was 95% there. This is basically to ensure every data will come with the labels that will be used to route the alerts dynamically.
These could be:
 - environment
 - team
 -  cluster
 -  application

**Re-org the receivers and channels**
Based of the metadata available, organise carefully the receivers and channels.
From multiple channels you can end up having only one!!!

**Create a template in alertmanager**
The template will spit out the raw value or transformed depending on labels so here is when you take the decision on how to tackle it.

### Setting up the solution, hands on.
This is a simple and quick example, this can go further than this.

1. **Populate** your alertmanager **gotpl** using existing data.
```
    {{ define  "env" }}
    {{-  if (index .Alerts  0).Labels.env  -}}
    {{- (index .Alerts  0).Labels.env  -}}
    {{-  end  -}}
    {{ end }}
```
2. **Group your alerts** per environment!!!!
This will ensure your template ranging will pick up only one in a kind environment.
```
group_by: [ env]
```
3. **Modify your receiver** accordingly to reflect the use of the template.
```receivers:

- name: YourReceiver
	slack_configs:
	- channel: '#alerts-to-env-{{- template "env" . -}}'
```
This is effectively one receiver to rule them all. 

5. **Normalize** the data among your alerts, prometheus rules.
6. **Test it! :)**
Use alertmanager metrics to see whether there was some issue sending alerts :) Or logs :)
```
alertmanager_alerts_invalid_total
```
