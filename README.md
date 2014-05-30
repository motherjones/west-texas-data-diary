Data Diary for "Is There a Risky Chemical Plant Near You?"
=====================

A data diary for our project on why we can't predict the next West, TX explosion

## The goal

In the wake of the April 17, 2013 explosion at a fertilizer plant in the town of West, Texas, we wanted to find where else plants posed a chemical risk to their surrounding communities. Read the published story [here](http://www.motherjones.com/environment/2014/04/west-texas-hazardous-chemical-map), which talks about the project methodology.

This data diary (inspired by [Dan Nguyen](https://github.com/dannguyen/bts-transstats-t100-domestic-demo)) walks through the steps of preparing the [EPA's Risk Management Plan](http://www2.epa.gov/rmp) data for mapping.

This guide will cover how to read a dataset about US chemical facilities that report to the EPA every five years, add geographical data, and run some basic analyses on it that will help you think about visualization.

## Before you get started

Here's what you'll need for this exercise:

### The data source

- EPA's Risk Management Plan data - The data for this project comes from the Center for Effective Government's Right-to-Know Network, which [maintains a database](http://www.rtknet.org/db/rmp) of the EPA's Risk Management Plan (RMP) reports, obtained through a public records request. Download a copy of the dataset we started with from this repo.

### Tools

- Microsoft Excel
- Your text editor of choice (My current preference is [Sublime Text](http://www.sublimetext.com/))
- [Texas A&M University's GeoServices](http://geoservices.tamu.edu/) - a low-cost geocoding service for large datasets. Includes lat/lon coordinates and census data.
- [Sequel Pro](http://www.sequelpro.com/) - A free MySQL client (for Mac OS X)
- [Papa Parser](papaparse.com) - a Javascript library that will help us reshape the facilities data
- [Our data reshaper](https://github.com/motherjones/mojo-news-apps-2014/tree/master/4/west-texas/rmp-transpose) - a script I built to reshape the facilities data


### The data you'll end up with (for cheaters)

- [RMP data geocoded and transposed](https://github.com/motherjones/mojo-news-apps-2014/blob/master/4/west-texas/rmp-transpose/facilities_remapped.tsv)

### What you should know about the data

The Environmental Protection Agency monitors about 12,000 chemical facilities under its Risk Management Plan. RMP facilities file a report with the EPA every 5 years. While you can look up individual RMP facilities, the entire dataset is only available through a public records request. The Center for Effective Government's Right-to-Know Network files a FOIA request each year in May and stores the data in its own database. The data here is up to date as of May 2013. Some facilities in this dataset may have been de-registered since the data was obtained.

Here's some data about the data:
- Number of records: about 61,166
- Number of fields: 14

The [EPA's description]() of the RMP data:

> Risk Management Program
>>Requirements
>>There are three levels of regulation under the Risk Management Program. Program 1 is for firms that have relatively safe processes and low risk. Programs 2 and 3 are more highly regulated than Program 1. These Programs are meant for firms that have a higher risk of affecting the public in case of a chemical spill. All facilities that are listed under the Risk Management Program must complete a hazard assessment (consisting of an off-site consequence analysis and five-year accident history), implement an emergency response program, and submit a Risk Management Plan (RMP) to EPA. Most firms must also create a detailed accident prevention program that will help to prevent the accidental release of hazardous chemicals.

>Four Sections of the Risk Management Program

>>Offsite Consequence Analysis
>>In this section of the program, firms outline their worse-case and alternative accident release scenarios. The worse-case scenario is an unlikely scenario that describes the potential consequences of the release of the largest single vessel containing a regulated substance that produces the greatest offsite endpoint distance. The alternative scenario describes a more likely scenario for a release that could affect the public.

>>Five-Year Accident History
>>A facility’s five year accident history describes all those accidents that have caused deaths, injuries, evacuations, sheltering-in-place, significant on-site or off-site property damage, or environmental damage significant on-site or off-site damage. As these criteria generally are associated with only the most serious accidents, many firms have accidental releases that are not reportable under the Risk Management Program. Only the most serious accidents are reported.

>>Prevention Program
>>The RMP prevention program requirements are similar to the requirements of the Occupational Safety and Health Administration Process Safety Management (PSM) standard program. Most RMP facilities must conduct operator training, implement written operating procedures, maintain equipment, and take other accident prevention measures.

>>Emergency Response Program
>>In addition to the requirements listed above, all facilities must work with their Local Emergency Planning Committees and other local responders to ensure that local responders are prepared to respond to emergencies at the facility. Facilities must also have mechanisms in place to notify local responders when a release occurs. Facilities that choose to use their own employees to respond to accidental releases must implement additional emergency response program measures, including having an emergency response plan, emergency equipment procedures, documentation of first aid and emergency medical treatment needed to treat chemical exposures, and trained emergency responders.proper measures are taken when accidents occur.


## Step 1: Download the RMP facilities data

We obtained the RMP data directly from the Center for Effective Government, in the form of an Excel file that was emailed to us. 

We have included that data in this repo, which you can download above. Your browser will download a __xlsx__ file weighing roughly 5.3 MB and it will be named: `RmpProcessChemsNew.xlsx`.

Opening this file will boot up Microsoft Excel. I've tried importing the file into Google Spreadsheets, but the file size slowed down the program to where I switched to Excel instead.


## Step 2: Run a quick data quality check

A couple of tips before you dive into the data, to save time and headache later:

- You are working with address data, which means there may be spaces and special characters you should clean up before importing into a SQL program. To do this, I recommend saving a copy of the original data, then search for and deleting characters like `\` and `"` in the `STREET` column. This will help you avoid scrambling the SQL import or geocoding down the road.

- Notice that in the first row of the spreadsheet, or the field headers, there is a space separating each word. SQL database managers (or at least ones I've worked with), do not like spaces in the header rows. Replace the spaces with underscores or camelCasing. For this exercise, let's use underscores.


## Step 3: Reshape the data.

Now that you're looking at the RMP dataset, you'll notice that each row represents a unique chemical reported at a given facility, not a unique facility. For example, there are 2 rows referring facility 100000000045 and facility 100000000054, for each unique chemical:

FacilityID | FacilityName | NameOfChemical
 ----------------|--------------|----------------:
 100000000045 | Yellow Breaches Water Treatment Plant | Chlorine
 100000000045 | Yellow Breaches Water Treatment Plant | Public OCA Chemical
 100000000054 | Sooner Cooperative, Inc | Ammonia (anhydrous)
 100000000054 | Sooner Cooperative, Inc | Public OCA Chemical


What we want to map, though, is each unique facility, and we want to display the list of all chemicals reported by that facility in a tooltip. In order to do that, we need to reshape the data so that each row represents a unique facility. I couldn't find a good way to do this through SQL or R or Pivot Tables, so with help from a few generous friends (thanks, [Noah](https://twitter.com/veltman)!), I [wrote a script](https://github.com/motherjones/mojo-news-apps-2014/tree/master/4/west-texas/rmp-transpose) that loops through the data set and creates one row per facility. For any repeated facilities, the program adds the new chemical into the chemicals column, separating each unique chemical with a semicolon within.

The result looks something like this:

 FacilityID | FacilityName | NameOfChemical
 ----------------|--------------|----------------:
 100000000045 | Yellow Breaches Water Treatment Plant | Chlorine; Public OCA Chemical
 100000000054 | Sooner Cooperative, Inc | Ammonia (anhydrous); Public OCA Chemical

Pretty magical. Shall we give it a whirl?

- Save the `RmpProcessChemsNew.xlsx` as a CSV and give it a name that tells you, "This is the version of the data that I cleaned up." In this case I'll call it `RmpProcessChemsNewClean.csv`. Detailing this in your file name can get unwieldy—at one point I named a file `rmp2andOr3allGroupedByFacilityAndChemIDgeoTotalAccidentsClean`-but they can also help you keep track of how you've been working with the data. It's probably not The Best Way to do version-control, but it worked in this case.

- Open [`parseFacilityChemicals.html`](https://github.com/motherjones/mojo-news-apps-2014/blob/master/4/west-texas/rmp-transpose/parseFacilityChemicals.html) in a web browser (preferably Chrome) and upload your new CSV.

- Voila! A new, reshaped, tsv file will download onto your machine.

> I should disclose that the need to reshape the data didn't occur to me until I was already pretty far (many months) into this project. But the smart way would have been to reshape first. So that's how I'm recommending you do it here. The hindsight would have saved me a lot of time and hassle.


## Step 4: Geocode your data

- Since we want to map this data, open your [reshaped tsv file](https://github.com/motherjones/mojo-news-apps-2014/blob/master/4/west-texas/rmp-transpose/facilities_remapped.tsv), and save the file as a CSV by clicking File > Save As > and choosing the comma separated values format in the filetype drop-down menu.

- The first 10 rows of that CSV look something like this:

Facility_Name | Process | Street | City | State | Zip | Latitude | Longitude | Accidents | Total_No_Chemicals | Chemical_Names
----------------|--------------|----------------|----------------|--------------|----------------|----------------|--------------|----------------|----------------|--------------:
Foster Farms  C St. (10/09 RMP Rev.) " " "" | 3 | 520 C Street | Turlock | CA | 95380 | 37.491 | -120.849 | 0 | 2 | Public OCA Chemical; Ammonia (anhydrous)
INITIATIVE FOODS, LLC | 2 | 1117  K Street " " "" | Sanger | CA | 93657 | 36.700701 | -119.551613 | 0 | 2 | Public OCA Chemical; Ammonia (anhydrous)
REC Silicon | 3 | 3322 Road  N N.E. " " "" | Moses Lake | WA | 98837 | 47.135278 | -119.193333 | 0 | 3 | Public OCA Chemical; Silane; Flammable Mixture
Moses Lake Plant # 80546 | 3 | 3245  N N. E. " " "" | Moses Lake | WA | 98837 | 47.133273 | -119.188444 | 0 | 2 | Public OCA Chemical; Ammonia (anhydrous)
Butterfield Water Treatment Plant | 3 | 1306 W.  B Street " " "" | Pasco | WA | 99301 | 46.220278 | -119.1025 | 0 | 2 | Public OCA Chemical; Chlorine
Big  L Packers " " "" | 2 | 12901 Packing House Road | Edison | CA | 93307 | 35.348056 | -118.867222 | 0 | 3 | Public OCA Chemical; Ammonia (anhydrous); Chlorine
Endicott, WA 384 | 2 | 215  E Street " " "" | Endicott | WA | 99125 | 46.927778 | -117.686389 | 0 | 3 | Public OCA Chemical; Ammonia (anhydrous); Ammonia (conc 20% or greater)
Big  N Fertilizer " " "" | 2 | 1201 SW 2nd Street | Tulia | TX | 79088 | 34.535556 | -101.778611 | 0 | 2 | Public OCA Chemical; Ammonia (anhydrous)
HOBART NH3 | 2 | 13160  8 ROAD " " "" | PLAINS | KS | 67869 | 37.291111 | -100.5225 | 0 | 2 | Public OCA Chemical; Ammonia (anhydrous)

- Go to [TAMU GeoServices](http://geoservices.tamu.edu/) and sign up for a free account.

- Follow the steps for a [Batch Geocoding](http://geoservices.tamu.edu/Services/Geocode/BatchProcess/) request. You'll get to choose what types of geodata you want back: coordinates, census tracts, county codes...a glorious selection, really.

- It will take anywhere between 1 day and a few days for TAMU to geocode data about this size. Typically, they send you an email when the job is done, but it's always good to follow up. You can download the results after logging into your TAMU account. Download and name the results `facilitiesRemappedGeocoded.csv` or some such.


## Step 5: Filter out the facilities you want to map

For this project, we initially wanted to map the facilities falling under Process 2 and Process 3. [Read about why](www.motherjones.com/environment/2014/03/west-texas-hazardous-chemical-map), but you could map all facilities and skip this step and go straight to mapping. I'll show you how I filtered the data anyway.

- I used [Sequel Pro](http://www.sequelpro.com/) as my SQL database manager. It has a simple interface and can handle large data files.


- 1. Boot up your MySQL server. Here's a [basic tutorial](https://www.digitalocean.com/community/articles/a-basic-mysql-tutorial) to install MySQL, but don't be too shy to ask for help. David Herzog and the kind folks at IRE/NICAR have a great tutorial for this. Open Sequel Pro. I usually login under the Socket using the username `root`, as shown below:

<img src="https://cloud.githubusercontent.com/assets/1892121/2896158/fef270e8-d568-11e3-9b01-c359ff0618a1.png"/>

<img src="https://cloud.githubusercontent.com/assets/1892121/2896157/fef0cbd0-d568-11e3-8b75-823d5c552903.png"/>

- 2. __Create a database__ - In your database manager, create a database named `rmpAll` or whatever you want to name it. In Sequel Pro, you'll go to Database > Add Database, and name it `rmpAll` or whatever you want to call it.

<img src="https://cloud.githubusercontent.com/assets/1892121/2896156/feee3e88-d568-11e3-8533-8d9f6175bde0.png"/>

- 3. __Create a table__ - Go to File > Import. 

<img src="http://assets.motherjones.com/interactives/projects/2014/4/westTexas/readmeImgs/Screen Shot 2014-05-01 at 2.05.07 PM.png"/>

Choose your geocoded CSV file, in this case `facilitiesRemappedGeocoded.csv`.

<img src="https://cloud.githubusercontent.com/assets/1892121/2896152/fee1da58-d568-11e3-94d4-418fee8ea2af.png"/>

Here is where you tell the database manager how to treat each field. Labels, text, descriptions, addresses, are usually treated as `VARCHAR`. Keep in mind that `VARCHAR` has a 255-character limit. Anything you want the database manager to treat as a number, such as pounds, dollars, and any other values you might add, subtract, divide, multiply, or perform math on, should be treated as `INT`.

<img src="https://cloud.githubusercontent.com/assets/1892121/2896151/fee1697e-d568-11e3-89bb-acb01bef4d3f.png"/>

Don't forget to Name the table in the upper right-hand corner. I'll call it rmpAll, since I'm importing the whole database. Click Import.

- 4. We'll write a simple query to filter so you only get the facilities that have Process (the EPA calls it Program) 2 and 3 (column name `Process`), and also those that have not been deregistered (column name `Is_Facility_Deregistered`). We'll also store the results of the query in a new table within our database:

`CREATE TABLE rmp2and3registered
SELECT *
FROM rmpAll
WHERE (Process = "2" OR Process = "3") AND Is_Facility_Deregistered = "n"`

If you only want to display a few columns in the results, replace the * in the `SELECT` line with the column names, separated by commas.

Run the query.


## Step 6: Export your filtered results as a new CSV

Your new table with the filtered results should show up in the Tables panel on the left-hand side. Right click on the new table then select Export... > As CSV file. (Don't mind all the table names in the screenshot below. Like I said, this tutorial benefits from hindsight; the actual process took months of trial and error.)

<img src="https://cloud.githubusercontent.com/assets/1892121/2896153/fee1da94-d568-11e3-8601-e15ad02792d9.png"/>

## Step 7: Time to map (on your own)!

Congratulations! Now you have a CSV, of the registered EPA facilities under Programs 2 and 3, with geodata of your choice. Now upload this to your mapping software of choice, and go forth! Visualize away.
