---
title: Engineering Custom Labeling Software
description: My Experience Implementing a Manufacturer Labeling Software System
date: 2021-03-24
tags: posts
layout: layouts/post.njk
---

In Fall and Winter 2020/21 I completed a contract to develop a suite
of applications for a manufacturing company. It was a small team and I reported
directly to a VP and worked with the floor manager to define requirements.
The work was primarily remote with a half a dozen visits to the office for
requirement elicitation and production integration. The nature of the small team
meant that my project was almost entirely self-supervised.

## Phase 1: Requirements Definition

I met with them and they showed me around the manufacturing floor and gave
me a high level overview of what they needed out of the project. They had
a label printer and barcode scanners. They needed automated label printing for
each unit manufactured (thousands per month). Units would be scanned as they
were stacked on their shipping pallets. Logs would be kept for each pallet
with their contents. Ideally automated shipping labels would be created that
listed the contents of each pallet. The primary application interface for the floor
staff should be scanning labels with a barcode scanner. To minimize risk we
decided to prioritize getting a minimum viable product (MVP) ready for a test
run so that we could prioritize efforts efficiently.

## Phase 2: Analysis of Prior Work

They had a previous engineer who had worked on this project but it was incomplete
and had never been integrated into their production workflow. I dove into their
codebase. It consisted of a few Python apps with most of the code being GUI code
using Qt for Python. It used MS Access hosted on a network drive as a
database. I decided to stay away from the GUI code for the beginning and get the
logic set up. The data models were good so I used them. He had
worked with their Excel guy to automate a macro to convert their job spreadsheets
to CSV files that made it easy to read in the required jobs into the database.
This code was solid so I incorporated it.

## Phase 3: Label Generation

The biggest uncertainty for me at the beginning of the project was automating
the generation of the labels. It turned out they had tried using ZebraDesigner
Pro which connected to the MS Access database to generate labels. But the software
key cost $345 per computer and they had misplaced their access key. So I researched
alternatives and determined that Reportlab Open-Source could be used to minimize
their cost and remove the headache of obtaining and setting up expensive license
keys for any computer that needed to print labels. Thanks to Lorenzo Simonassi
for sharing [this Reportlab code snippet](https://stackoverflow.com/a/38507014)
which didn't require much tweaking for the final label generation process.

I coded up a quick text-based user interface (TUI) to select from scheduled jobs.
From there the user could specify all labels for a job, a single label, or all
labels for a series of line items. The program then queried the database for the
required data then a PDF containing all of the labels would be generated and
opened using the Python code: `os.system("start " + LOCAL_LABELS_FILE)`. The user
then reviews and initiates the print from their PDF client.

![Example label](/img/label.jpg)

Later on I engineered a solution to dynamically decrease the label font size in
the case of long text fields to ensure that all text is displayed on each label.
This was easy to do using the behavior of `Frame.add()` defined [here](https://github.com/Distrotech/reportlab/blob/master/src/reportlab/platypus/frames.py)
which returns `0` if the flowable to be added does not fit, and `1` if the draw
was successful. In this case we simply define a label generating function
`generate_label_with_font_size(self, font_size, canvas, label_data)` that takes
in a font size and returns `False` in the case of any failed `add()` and `True`
otherwise. A wrapper function begins with the baseline font size of your choice
and decrements the font size until `generate_label_with_font_size()` succeeds.

```python
def generate_label(self, canvas, label_data):
    font_size = 14
    while font_size > 0:
        if self.generate_label_with_font_size(font_size, canvas, label_data):
            break
        font_size -= 1
```

## Phase 4: Job Scheduler

I forwent the incomplete GUI that had been implemented previously for a simple
TUI again in the interest of time. All the scheduler UI had to do was get the
job-to-be-scheduled's CSV file and have the user specify the ship date for the job.
So a simple drag-and-drop of the CSV file from MS Windows' File Explorer to the
TUI pasted the full file-path to the program's stdin. Then the program would
prompt the user for the job's ship date. I was able to use prior code to insert
the job data into the database.

![Scheduler TUI](/img/admin.jpg)

## Phase 5: MS Access Issues

During Phase 4 I noticed that MS Access did not support concurrent writes. I
did some quick searching and learned that MS Access was not recommended for
production workflows. It struggles with concurrent accesses and network hiccups.
It is also prone to corruption. Even an infrequent number of database issues could
be a huge headache in the future.

So I migrated the database schema to an MS Azure-managed PostgreSQL instance and
updated the database adapter code to use psycopg. This worked much better.

## Phase 6: Scanner Application

The barcode scanner hardware simply writes to stdout by default. So I coded up
another simple TUI to read in the barcode labels sent to stdin by the scanner.
For the MVP I decided not to develop a full-on GUI, and provide audio user feedback
for successful vs unsuccessful actions.

To define user interface actions,
I used MS Word to create printable barcode documents
[per this](https://support.microsoft.com/en-us/office/field-codes-displaybarcode-6d81eade-762d-4b44-ae81-f9d3d9e07be3).
To assign a unit to a pallet,
the user scans the pallet's barcode then the unit being stacked. To unassign,
the user scans a barcode labeled as "Unassign Unit" followed by the unit's
barcode. To create a new label for a new shipping pallet,
the user scans the "New Pallet" barcode followed by a panel belonging to the
same job as the new pallet.

The program would emit a failure beep upon defined warnings/errors including:
database connection errors, double-assignment of a unit and assignment of a unit
to a pallet belonging to a different job. It would emit a success beep upon
successful assignment of a unit to a pallet.

## Phase 7: Scanner Application Additions

We quickly determined that it was worth the effort for me to develop a GUI
to display pallet contents and job info. This would help the floor workers
keep track of pallet contents to minimize costly missed scans.

Furthermore, I implemented automatic packing slip printing which listed job
information and pallet contents. The information was pulled from the database
and then the program uses openpyxl to populate an Excel template. When a job was
marked as complete, a master list would be printed listing all
units in the job and which pallet they were stacked on. The scanner app initiated
the print job without a user needing to use the computer to navigate the Windows
print menu using the following code which sent the file to the computer's default
printer:

```python
win32api.ShellExecute (
    0,
    "print",
    filename,
    '/d:"%s"' % win32print.GetDefaultPrinter (),
    ".",
    0
)
```

The GUI was implemented with Tkinter. It used a simple grid layout with
4 columns one for each pallet . It kept a log of the scanning actions at the
bottom. It used a simple colored square to indicate application status with
the color green indicating error-free and the color red indicating an error
with an error-message below. Pallet unit counts were emphasized in large bold
font at the top of each column with the hope that workers could use that to
minimize missed scans. The GUI was displayed on a large television
above the stacking station.

![Scanner screenshot](/img/scanner.jpg)

I then incorporated production statistic generation functionality into the
admin app. This broke down the number of stacked units broken down by hour,
day and month with an hourly stacking rate for each.

## Phase 8: Integration and Bug-Fixes

I used PyInstaller to bundle the apps together into a single EXE file. This
was pretty straightforward, except for the apps requiring ReportLab for which
[hidden imports](https://pyinstaller.readthedocs.io/en/stable/when-things-go-wrong.html#listing-hidden-imports)
needed to be specified for each ReportLab submodule.

The biggest problem initially was the network reliability on the computer
running the scanner app. It was connected via WiFi and periodic network
disconnects were common. This would break the database connection in the scanner
app. I coded up an error message for the network issue and placed a button
on the GUI to attempt a database reconnect. They also hardwired the network
connection via ethernet which reduced network disconnects drastically.

Another issue that came up was lost labels. During the manufacturing process,
a unit would occasionally lose its label. We could determine which job and
line item the unit belonged to but we didn't know the unit's ID value for
reprint of its label. So I had to create workaround code in the scanner app logic
to flag for unit count disparities rather than expecting each unit ID to be
accounted for.

## Lessons and Takeaways

Network is unreliable! Accounting for network disconnects in my code was not
something I had thought of beforehand but it proved to be essential.

Visuals are important! Having pallet contents listed on the television in real-time
and clearly visible to those stacking the units allowed the workers to keep
track of what they were doing and minimize misscans.

The beginning of joining an existing team was quite challenging as I had
to catch up to the common conception of the project space. It was important to
be patient and trust that with each working hour
researching the project space, I was making
effective progress towards the end goals even if I wasn't writing code in the beginning.

When jumping into a new team and project, vocabulary definition is one of the
first steps in understanding the project and being able to communicate with the
team. The team used jargon that I was not
familiar with and so I had to ask many clarification questions in the beginning.
The team members had minimal experience managing custom software and so I had
to be careful to translate my considerations to outcomes that were tangible to
them rather than try to communicate implementation details that would be troublesome
to effectively communicate.

Being the only team member working remotely is not easy. It was a small team
and each member had a large number of responsibilities. In the beginning I would
send long emails asking questions. Responses were understandably inconsistent.

It was not until after diving into the previous code and orienting myself to the
organization's documents and workflows by exploring their network drive that
I was able to formulate my questions well and get the needed information from
an efficient hour and a half team meeting. Had I been working in the office it
would have
been easier to ask more questions as I went which may have sped up the process.

Top-down learning works, especially for a project such as this in which the
functionality is straightforward and the project scale is small. My Python
experience was limited to small projects for undergrad Security and Web Systems
courses. This baseline proficiency was complimented by my stronger C++ and
JavaScript experience which gave me reference points for design patterns and
in understanding and searching for tool-specific features. In this project I
used goals and functionalities to guide my learning as I implemented. As my
application code became too unwieldy, I would allocate time for a refactor.

## What I'd Do Differently

In hindsight I would have doubled my
workload estimates. In the beginning of the project I fell behind my predicted
milestone timeline. This was a result of overestimating the functionality of the
previous engineer's work. I would be better served had I been cynical about the state
of the untested code.

Overall, I consider this project to have been efficiently and effectively implemented.
The MVP approach worked well as we were able to get an MVP ready for testing quickly.
Subsequent testing gave us a much better conception of which additional features
to prioritize.
