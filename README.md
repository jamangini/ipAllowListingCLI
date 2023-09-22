# 2022 repo README

# ipAllowListingCLI
Allow List IPs on LoginIpRange object for Salesforce User Profiles. [Announcement and demo can be found here](https://salesforce-internal.slack.com/archives/C01S1KE15AL/p1660607613061249?thread_ts=1660607483.161429&cid=C01S1KE15AL).

## What are we trying to accomplish?
The natural evolution of [my older repository](https://git.soma.salesforce.com/jmangini/IPWhitelistingCLI), I wanted a tool for Salesforce Adminstrators to apply LoginIpRanges with an API. The other options are:
- apply the LoginIpRanges manually, through the UI
- prep a CSV file and use Data Loader or Workbench to run an INSERT on the User Profile, which takes a number of steps

## A WORD OF CAUTION
Using this tool will apply a set of LoginIpRanges, and ***remove IPs that aren't in the XML***, so if you run this tool, keep a very close eye on the Setup Audit Trail to ensure you didn't just clobber pre-existing configuration that was set in the past

## Pre-requisites for running this tool
- [SFDX](https://developer.salesforce.com/tools/sfdxcli)
- [jq](https://stedolan.github.io/jq/)
- [dw](https://github.com/mulesoft-labs/data-weave-cli)
- [tree](https://formulae.brew.sh/formula/tree) (optional)

## Instructions
1. adjust the SFDX query as needed, in the `build.sh` file
2. cd into the `Archive/` directory, and run `../build.sh`

## A walkthrough of the components in this repo
ipAllowListingCLI/ <br>
├── Archive <br>
&nbsp;│&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;└── profiles <br>
├── README.md <br>
├── build.sh <br>
├── profileToPackage.dwl <br>
└── templateProfile.xml <br>

### Metadata API directory structure
`Archive ==> profiles` will serve as the container that the script puts files into. `package.xml` gets written under `Archive/`, and any User Profile returned in the SOQL command will get written to `Archive/profiles/`.

### DataWeave translation script (`profileToPackage.dwl`)
The fundamentals of the Salesforce Metadata API (MDAPI) is to have artifacts denoted in the package.xml, for whatever is in the ZIP file to be deployed. [This script](https://git.soma.salesforce.com/jmangini/ipAllowListingCLI/blob/main/profileToPackage.dwl) will generate a package.xml which will look like the following: <br>
```XML
<?xml version='1.0' encoding='UTF-8'?>
<Package xmlns="http://soap.sforce.com/2006/04/metadata">
  <types>
    <members>member</members>
    <name>Profile</name>
  </types>
  <version>48.0</version>
</Package>
```
The biggest difference (*and a TIL for yours truly*) is that the MDAPI allows both single quotes and double quotes (I assumed that a failure would take place if one strayed from the templates [denoted in the documentation](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/manifest_samples.htm))

If you've never used DataWeave, MuleSoft provides [a handy playground](https://developer.mulesoft.com/learn/dataweave/) to test this incredibly powerful programming language.

### Template User Profile (`templateProfile.xml`)
As the build script runs, it copies [the same forty six line file](https://git.soma.salesforce.com/jmangini/ipAllowListingCLI/blob/main/templateProfile.xml) as newly named `.profile` files. If you want to use this repository, and apply IP Ranges of different types, you would update this template, and run the script.

### The bash script (`build.sh`)
Let's step through each portion of [the script](https://git.soma.salesforce.com/jmangini/ipAllowListingCLI/blob/main/build.sh) and talk about what they do. <br>
#### 1.) The `SFDX` command
`sfdx force:data:soql:query -q "SELECT Name From Profile WHERE Name LIKE 'proofOfConcept%'" -u alohaDev -r json` <br> <br>
Running a simple SOQL command for a given user alias (`-u`) and having the returned result be in a JSON format. If you want to customize which User Profiles to query, you would update the statement inside of the double quotes. The data returns as a JSON blob, which will look something like this:
```JSON
{
  "status": 0,
  "result": {
    "done": true,
    "totalSize": 2,
    "records": [
      {
        "attributes": {
          "type": "Profile",
          "url": "/services/data/v55.0/sobjects/Profile/00eg0000000IbvkAAC"
        },
        "Name": "proofOfConcept"
      },
      {
        "attributes": {
          "type": "Profile",
          "url": "/services/data/v55.0/sobjects/Profile/00eg0000000IbvpAAC"
        },
        "Name": "proofOfConcept_deux"
      }
    ]
  }
}
```
What we really only care about, is the *value* that is tied to each `Name` *key*, for each of the entries in the *records* array. So that leads us into... <br>
#### 2.) The `jq` command
`jq -r '.result.records[].Name' > ../listofUserProfiles.txt` <br> <br>
Check out the explanation of jq in the linked pre-req up above, it is an incredibly valuable CLI tool. It takes ^^^ the JSON blob, and distills down just the `Name` values into a text file, which we pull from later. `-r` just removes the double quotes - there's also [a playground for this library](https://jqplay.org/), if you want to familiarize yourself with the tool.
#### 3.) The bash command to create *pro-files*
`cat ../listofUserProfiles.txt | while read -r line; do cp ../templateProfile.xml ./profiles/$line.profile; done` <br> <br>
The jq command up above creates a text file with Profile Names. These three commands simply pipe the output of that file to a handy while() loop that creates a file from the template XML skeleton.
#### 4.) The DataWeave CLI command
`dw -i payload ../listofUserProfiles.txt -f ../profileToPackage.dwl > package.xml` <br> <br>
The documentation for this library could use some work (*this one-liner I needed more help than other parts of this*), but it essentially has a payload file (`-i`), and then the translation script (`-f`) and then outputs to the target `package.xml` file.
#### 5.) ZIP up all this data
`zip -r -X Archive.zip *` <br> <br>
A macOS specific command to ZIP up all the files created, so that a SOAP deploy can occur.
#### 6.) write to console, run tree, and perform deploy
`echo "and now a tree command to show you what you've made" && tree .. && sfdx force:mdapi:deploy -w 2 -u alohaDev -f ./Archive.zip -s --soapdeploy` <br> <br>
The shell script creates files, and using `tree` just helps to denote what gets created, as a result it helps to just show the resulting directory structure, and files created.
![Terminal output from running the build.sh script](./build_Output.png "output of build.sh")

## Notes
Running `git status` will tell you the files created, to `rm` || cleanup after executing the script.
![Terminal output from executing 'git status'](./git_Status.png "output of git status")
