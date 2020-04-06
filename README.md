
<!-- README.md is generated from README.Rmd. Please edit that file -->

# labelR - Pairwise Data Labeling on Mechanical Turk

<!-- badges: start -->

<!-- badges: end -->

labelR is a package that assists users in conducting data labeling with
pairwise comparisons on Amazon’s Mechanical Turk (MTurk). This package
replicates and extends the functionality of the sentimentIt package (now
deprecated) to interface directly with MTurk through R, rather than
through the sentimentIt servers.

## Installation

<!--
You can install the released version of labelR from [CRAN](https://CRAN.R-project.org) with:

``` r
install.packages("labelR")
```
-->

You can install the development version from
[GitHub](https://github.com/) with:

``` r
# install.packages("devtools")
devtools::install_github("RydenButler/labelR")
```

## Tutorial

This tutorial provides a complete walkthrough of how to use labelR to
conduct pairwise data labeling on MTurk. It assumes that you have both a
basic level of understanding about both MTurk and R. The tutorial begins
by demonstrating how to create a worker qualification and corresponding
qualification test that will permit only authorized workers to complete
your assignments (MTurk documentation will refer to these as “human
intelligence tasks” or “HITs”). Next, it demos how to format a HIT
template and pass the formatting to MTurk to use with your assignments.
Finally, it shows how to send assignment to MTurk for labeling, retrieve
the labeled results, and check worker quality.

### Before using labelR

labelR depends on the pyMTurkR package. The package allows for R to
interface with MTurk through using python code. As such, you will need
to follow the more-complicated-than-usual installation instructions
provided in [its github repo](https://github.com/cloudyr/pyMTurkR) so
that R and python will play nicely together.

### Getting started

First, load labelR and pyMTurkR into your session and import your MTurk
credentials from a separate file.

``` r
library(labelR)
library(pyMTurkR)

source('MTurkCredentials.R')
```

I store my credentials in the file MTurkCredentials.R, which is
formatted as follows:

``` r
access_key = "ABC123"
secret_key = "Shhh!"
```

Store your access key and secret in the R session so that they can be
accessed by pyMTurkR throughout. For this tutorial, and while testing
out the system, you should remain in sandbox mode by keeping
pyMTurkR.sandbox = T. You should always run pyMTurkR::AccountBalance()
before sending HITs to MTurk to verfiy whether you are in sandbox mode.
If you can execute the code below without error, you are ready to begin

``` r
Sys.setenv(AWS_ACCESS_KEY_ID = access_key)
Sys.setenv(AWS_SECRET_ACCESS_KEY = secret_key)
options(pyMTurkR.sandbox = T) # Change sandbox = F when ready to run on MTurk
pyMTurkR::AccountBalance() # Test that your pyMturkR settings are correct
# When in sandbox mode, your balance will be $10,000.00
# When not in sandbox mode, your balance will reflect the pre-paid funds in your account
```

> ***A note on sandbox mode*** MTurk offers a sandbox mode for testing
> out your HITs before committing to pay for the work. There is both a
> requester sandbox, where you can test formatting and sending HITs, as
> well as a worker sandbox, where you can view your HITs as the workers
> will see them. This is helpful for testing out your qualification
> tests, which can only be fully rendered when accessing the link
> through a posted HIT.

### Making a Qualification with a Test

Qualification tests on MTurk serve the dual purpose of training workers
on how your assignments should be completed as well as barring workers
that may provide low quality work. Therefore it is important that the
qualification test contain a complete and accurate description of the
worker’s job along with examples that they can use to better their
understanding of nuances in the data labeling task. The qualification
test and its answer key are supplied to MTurk in XML format. To spare
users from writing XML code, labelR uses a series of functions whereby
users can pass in HTML-formatted strings which will be automatically
written to the XML files. Users should use the formatTest function with
apporopriately supplied inputs to automatically generate the XML files
for the qualification test and answer key.

Some of the inputs to formatTest can be unintuitive. An object
containing example inputs is provided as data in the labelR pacakge.

``` r
data("Text")
QualTest <- formatTest(title = Text$Title,
                       instruction_overview = Text$InstructionOverview,
                       instruction_list = Text$InstructionList,
                       question_prompt = Text$QuestionPrompt,
                       practice_overview = Text$PracticeOverview,
                       practice_questions = Text$PracticeQuestions,
                       test_overview = Text$TestOverview,
                       test_questions = Text$TestQuestions,
                       closing_message = Text$ClosingMessage)
```

The arguments title, instruction\_overview, question\_prompt,
practice\_overview, test\_overview, and closing\_message all require a
string object. The instruction\_list argument requires a nested list
object with a specific format. This content will create a bulleted list
of key rules and simple examples to explain the task to workers. If an
element of the primary list is a string, it is converted to a bullet in
the training module. If an element of the primary list is a list object,
the first element of this sub-list should be a string, which will be
converted to a bullet in the training module. The second element in the
sub-list should also be a list containing at least one string-formatted
element. The elements of this list will be converted to sub-bullets in
the training module, which are indented with respect to the top-level
bullets and will possess a different bullet style.

``` r
InstructionList <- list(
  'If both advertisements attack a candidate, pick whichever of the two advertisements is most negative',
  'If both advertisements praise a candidate, pick whichever of the two advertisements is least positive',
  list("Ads that attack a candidate's personal characteristics are generally more negative than ads that  attack a candidate's record or job performance",
       list('"Bob is dishonest" is more negative than "Bob is too liberal"')),
  list('Ads that attack an individual are generally more negative than ads that attack a group',
       list('"Bob is a Washington fat cat" is more negative than "We need to stop the fat cats in Washington"')),
  list('Ads that attack a specific candidate alone are generally more negative than ads that contrast two candidates', 
       list('"Bob is unqualified" is more negative than "Bob is unqualified, but Jill is very experienced"')),
  list('Ads that state a policy position are generally less positive than ads that praise a candidate as a person',
       list('"Bob will fix unemployment" is less positive than "Bob is a leader"'))
)
```

The above R object, when supplied to formatTest’s instruction\_list
argument, will be rendered in the qualification test as

> 
> 
> <ul>
> 
> <li>
> 
> If both advertisements attack a candidate, pick whichever of the two
> advertisements is most negative
> 
> </li>
> 
> <li>
> 
> If both advertisements praise a candidate, pick whichever of the two
> advertisements is least positive
> 
> </li>
> 
> <li>
> 
> Ads that attack a candidate’s personal characteristics are generally
> more negative than ads that attack a candidate’s record or job
> performance
> 
> <ul>
> 
> <li>
> 
> “Bob is dishonest” is more negative than “Bob is too liberal”
> 
> </li>
> 
> </ul>
> 
> </li>
> 
> <li>
> 
> Ads that attack an individual are generally more negative than ads
> that attack a group
> 
> <ul>
> 
> <li>
> 
> “Bob is a Washington fat cat” is more negative than “We need to stop
> the fat cats in Washington”
> 
> </li>
> 
> </ul>
> 
> </li>
> 
> <li>
> 
> Ads that attack a specific candidate alone are generally more negative
> than ads that contrast two candidates
> 
> <ul>
> 
> <li>
> 
> “Bob is unqualified” is more negative than “Bob is unqualified, but
> Jill is very experienced”
> 
> </li>
> 
> </ul>
> 
> </li>
> 
> <li>
> 
> Ads that state a policy position are generally less positive than ads
> that praise a candidate as a person
> 
> <ul>
> 
> <li>
> 
> “Bob will fix unemployment” is less positive than “Bob is a leader”
> 
> </li>
> 
> </ul>
> 
> </li>
> 
> </ul>

The practice\_questions argument also takes a nested list object with a
particular format. It contains the contents of example questions offered
as guidance for workers in the qualification test. The primary list
contains a series of sub-lists, one for each practice question. Each
sub-list is comprised of three elements. The first and second element
are strings, which contain the texts for documents 1 and 2 of a pairwise
comparison, respectively. The third element is a string containing
explanatory text that indicates which document the worker should select
and why that answer is correct.

``` r
PracticeQuestions = list(
  list("[Announcer]: Mark Udall claims he voted for tax cuts 65 times. 
       [Reporter]: \"Actually it's not true.\" 
       [Mark Udall]: \"I'm just kidding.\" Udall claims that now he supports our troops, drilling off shore, bi-partisanship. 
       [Mark Udall]: \"I'm just kidding.\" 
       [Announcer]: But through his career he has opposed them all. <font color=\"red\">Udall has a lot of ads, but a shortage of truth.</font> 
       [Mark Udall]: \"I'm just kidding.\" 
       [Announcer]: <font color=\"red\">Mark Udall is not honest about his past</font> and now it's catching up with him. 
       The National Republican Senatorial Committee is responsible for the content of this advertising. 
       [PFB]: NATIONAL REPUBLICAN SENATORIAL COMMITTEE  (9)",
       "[Announcer]: You want a leader you can believe in, who shares your values, and vision for the future. 
       On March 11th vote Erik Fleming for United States Senator. 
       He knows the importance and power of faith. 
       In these uncertain and changing times <font color=\"green\">Erik Fleming can guide us to a better more secure future for our families. 
       Believe in Erik Fleming. 
       He will lead us in the right direction.</font> 
       [Erik Fleming]: \"I'm Erik Fleming and I approve this message.\" 
       [PFB]: THE FLEMING FOR US SENATE CAMPAIGN COMMITTEE  (10)",
       "The correct answer is Ad 1. 
       It attacks the candidate in question, Mark Udall, by describing him as having \"a shortage of truth,\" and claiming he is \"not honest about his past.\" 
       Ad 2 is positive about the advertisement's subject, Erik Fleming. 
       It calls him \"a leader you can believe in,\" and promises \"he will lead us in the right direction.\""
  ),
  list("[Tobie Shramek]: \"Some folks in Congress thought that our soldiers on the front line deserved a salary bonus and that our veterans deserve first-class healthcare and educational opportunity. 
       <font color=\"red\">Roger Wicker voted against salary bonuses for our troops in Iraq and voted to cut veterans' benefits dozens of times. 
       Roger Wicker voted nine times to raise his own pay.</font> Let me repeat that. 
       Roger Wicker voted nine times to raise his own pay. That doesn't seem right.\" 
       [Announcer]: The Democratic Senatorial Campaign Committee is responsible for the content of this advertising. 
       [PFB]: DEMOCRATIC SENATORIAL CAMPAIGN COMMITTEE",
       "[Announcer]: Ronnie Musgrove's road to Washington? Proceed with great caution. 
       As Governor, <font color=\"red\">Musgrove recked the economy, lost 38,00 jobs and racked up a $700 million deficit. 
       Musgrove steered us into the Beef Plant Scandal, left us a $55 million bill.</font> 
       And his biggest contributors? <font color=\"red\">Indicted for attempted bribery. Failures. Scandals. Dead ends. 
       We just can't afford another wrong turn with Ronnie Musgrove.</font> 
       The National Republican Senatorial Committee is responsible for the content of this ad. 
       [PFB]: NATIONAL REPUBLICAN SENATORIAL COMMITTEE", 
       "The correct answer is Ad 2. 
       Though Ad 1 criticizes the candidate for having \"voted against salary bonuses for our troops,\" Ad 2 is more critical of its candidate. 
       It claims that Musgrove \"recked[sic] the economy\" and implicates him in the \"attempted bribery\" and indictment of his \"biggest contributors.\""
  )
)
```

The test\_questions argument requires an object of a format similar to
that used in practice\_questions. It contains the contents of test
questions used to evaluate whether workers pass your qualification test.
As before, the primary list contains a series of sub-lists, one for each
test question. Each sub-list is comprised of three elements. The first
and second element are strings, which contain the texts for documents 1
and 2 of a pairwise comparison, respectively. The third element is a
numeric integer indicating which element of the list is the correct
answer.

``` r
TestQuestions = list(
  list(
    "[Announcer]: Major newspapers across the state are endorsing Kay Hagan for U.S. Senate, calling Hagan a bundle of brains and energy. 
    A businesswoman, mother of three, Kay's ranked one of North Carolina's ten most effective senators. 
    With a moderate and business-friendly record, her priorities include fiscal responsibility, investing in education, healthcare reform and new energy development. 
    Kay Hagan. Exactly the change our economy needs. 
    [Kay Hagan]: \"I'm Kay Hagan, and I approve this message.\" 
    [PFB]: HAGAN SENATE COMMITTEE, INC.",
    "[Announcer]: Liberal Mary Landrieu: she supported amnesty for illegal immigrants, a plan that could have cost taxpayers 126 billion and require some immigrant workers to paid more than Americans for doing the same job. 
    Liberal Mary Landrieu voted to allow immigrants to collect Social Security benefits for work done illegally. 
    When it comes to immigration, Mary Landrieu is on the wrong side of the fence. 
    The National Republican Senatorial Committee is responsible for the content of this advertising. 
    [PFB]: NATIONAL REPUBLICAN SENATORIAL COMMITTEE",
    2),
  list(
    "[Jim Martin]: \"I'm Jim Martin, and I approve this message.\" 
    [Announcer]: Saxby Chambliss says Congress had to act, but why did they have to act? 
    It was Saxby Chambliss who voted to deregulate Wall Street speculators. 
    Saxby Chambliss took hundreds of thousands of dollars from Wall Street, and voted to let them do whatever they wanted. 
    He voted to give them billions more in tax breaks, and what happened? 
    A $700 billion taxpayer bailout, and a trillion dollar deficit. Saxby economics at the root of the problem. 
    [PFB]: MARTIN FOR SENATE &amp; DEMOCRATIC PARTY OF GEORGIA",
    "[Mark Warner]: \"I will not raise your taxes.\" 
    [Announcer #1]: Mark Warner broke his word and gave us the largest tax increase in Virginia history. 
    [Announcer #2]: Campaigning with Barack Obama, now Warner's talking even higher taxes, bigger spending and limiting domestic oil production, costing you more. 
    [Announcer #1]: We need Jim Gilmore in the U.S. Senate and so does John McCain. 
    Veterans who will keep America safe, keep taxes low and bring down gas prices. 
    [Announcer #2]: Principled. Honest. Jim Gilmore. 
    [Jim Gilmore]: \"I'm Jim Gilmore, candidate for the U.S. Senate, and I approve this message.\" 
    [PFB]: REPUBLICAN PARTY OF VIRGINIA",
    1)
)
```

When the correctly-formatted arguments are supplied to formatTest, the
function writes two XML files corresponding to the qualificaition test
code and the answer key code. These files are both returned from the
function call and written to your working directory as QualTest.xml and
AnswerKey.xml. The .xml files can be manually edited to change the
qualification test’s settings from its default format. The
FormattedContent sections of the QualTest file accept HTML-formatted
strings.

If you choose to manually edit the .xml files, you can re-load them into
R with the following code, provided that QualTest.xml and AnswerKey.xml
are still in your working directory.

``` r
TestQuestions <- paste0(readLines("QualTest.xml",
                                  warn = FALSE),
                        collapse = "")
TestKey <- paste0(readLines("AnswerKey.xml", 
                            warn = FALSE),
                  collapse = "")
```

Otherwise, if using the default files generated from formatTest, you can
access the files directly from the object to which formatTest’s output
is assigned.

With a properly formatted qualificaiton test test and answer key, you
are ready to create a qualification with the desired properties. The
below code accomplishes this using pyMTurkR::CreateQualificationType. It
take the following arguments: name, a string which provides a title for
the qualification that is visible to both you and to workers;
description, a string explaining the purpose of the qualification;
status, a string indicating whether the qualification should be active
(use this) or inactive; test and answer key, which take the
XML-formatted strings created from formatTest; test.duration, a numeric
integer that governs the time (in seconds) in which workers are allowed
to complete the test; and retry.delay, a numeric integer determining how
long workers have to wait to retake the test (when NULL, workers may not
ever retake the
test).

``` r
Description <- 'Qualifies workers to participate in Compare Senate Ads HITs'

QualificationWithTest <- CreateQualificationType(name = 'Compare Senate Ads',
                                                 description = Description,
                                                 status = 'Active',
                                                 test = QualTest$Questions,
                                                 answerkey = QualTest$Answers,
                                                 test.duration = 60 * 60, 
                                                 retry.delay = NULL)
```

### Making a HIT

Once your qualification has been created, you may proceed to creating
and formatting the HIT template that you will use to transmit pairwise
comparisons to workers. The HIT template is constructed with HTML code,
which can be automatically formatted using the formatHIT function.

``` r
HIT_code <- formatHIT(question_prompt = 'Which ad is most negative?',
                      short_instruction_list = ShortInstructions,
                      instruction_overview = InstructionOverview,
                      instruction_list = InstructionList,
                      closing_message = ClosingMessage)
```

The question\_prompt argument take a string that reminds workers of
their task. Remember, any workers participating in your HITs has
completed training through your qualification test, and so should be
familiar with the task at hand. As such, the question prompt can be
reduced from its form in the qualification test if desired. The
short\_instruction\_list argument take another specially-formatted list
object. The object’s content will render brief instructions in a
left-hand sidebar next to eahc pariwise comparison. The brief
instructions will appear as a bulleted list (of dashes) containing
simplified instruction or examples. Each element of the primary list is
a sub-list. The first element of the sub-list is a string that will be
rendered as a sub-title (a HTML h3 tag) in the list. this can be left
blank using empty quotes. Subsequent elements of the sub-list are
strings, each of which will receive a separate bullet (dash). An
example, also available in Text object found in data(“Text”), is shown
below.

``` r
ShortInstructions = list(
  list(
    '',
    'If both are negative pick the most negative',
    'If both are positive pick the least positive'
    ),
  list('What makes an ad more negative?',
       'It attacks a candidates character instead of their policy record',
       'It only attacks a candidate without comparing them to another candidate'
       )
  )
```

Finally, the instruction\_overview, instruction\_list, and
closing\_message arguments of formatHIT should take the same objects
used in formatTest. Workers may click a link in the sidebar of each HIT
that shows them a set of full instructions. Since you created detailed
instructions for your qualification test, you should repeat those here
so that workers receive consistent instructions throughout.

Similar to formatTest, formatHIT will return the HTML-formatted string
within R as well as write a HIT.html file to your working directory. You
can view the fully rendered HIT as it will appear to workers by opening
the .html file using a web browser. You can edit further edit the HIT’s
HTML code manually by opening the .html file using a text editor.

To create the HIT on MTurk, it is recommended that you sign into the
MTurk website as a requester, navigate to the Create tab, click New
Project, and select Item Equality from the left sidebar for the default
task setup. On the first page of the HIT setup you can specify how the
title, description, price, and duration of your HITs, as well as apply
your custom qualification by selectign it from the drop-down menu
populated by pressing the “(+) Add another criterion” button. For the
second page of the HIT setup, you should copy the complete contents of
your HIT.html file and paste it to fill the entire page of formatting
code on the page. The third page of the HIT setup will render your HIT
for previewing before confirming its creation. Note that this process is
just generating a template for your HITs, and that no HITs will be
posted for workers to complete at this point.

### Sending HITs to MTurk

Once you are satisfied with the template of for your HITs, you are ready
to begin sending HITs to workers for completion. Assuming you have a
corpus of documents ready to label and some principled way of determing
which documents should be compared, the sendComparisons function will
upload your HITs for workers to complete. The arguments hit\_type and
hit\_layout are alpha-numeric strings that are found by clicking on the
project name for your completed HIT template in the MTurk dashboard.
This can be reached by signing into the MTurk website as a requester,
navigating to the Create tab, clicking New Batch with and Existing
Project, and clicking the blue lettering under Project Name that
corresponds with your desired hit template. A box will appear with the
HITType ID and Layout ID. The comparisons\_to\_send argument takes a
matrix with two columns (one for each document) and a number of rows
equivalent to the number of comparisons you wish to upload for
completion. Each cell in the matrix should be a string that contains the
text of a document, with the two texts in each row of the matrix
constituting a single pairwise comparison. Executing the sendComparisons
function will return a vector of HIT IDs, one for each uploaded HIT. You
should store these IDs in R by assigning sendComparisons to an object,
as you will need these later for downloading your completed HITs from
MTurk.

``` r
n_docs <- 4 # number of example documents
n_comparisons <- 2 # number of comparisons
# create the documents
Documents <- data.frame(doc_ids = 1:n_docs,
                        doc_text = paste('test document', 1:n_docs))
# assign pairwise comparisons by doc id
CompareDocs <- matrix(sample(x = Documents$doc_ids,
                                    size = n_comparisons*2),
                             ncol = n_comparisons_per_doc)
# assign pairwise comparisons by doc text
Comparisons <- t(apply(CompareDocs, 1, function(x) Documents$doc_text[x]))

current_HITs <- sendComparisons(hit_type = '3W850VB9FSF0KVMX3N9YBOT3XV3K1W', # HIT type id from MTurk requester dashboard
                                hit_layout = '38X1F3X9FX9TJLGJJRSPLUQ2YXJY5C', # HIT layout id from MTurk requester dashboard
                                comparisons_to_send = Comparisons)
```

> ***A note on cancelling HITs*** Though it is recommended that you
> conduct thorough testing in sandbox mode before posting actual work to
> MTurk, accidents do happen. In the event of a catastrophic mishap, you
> can manually expire HITs that have been uploaded to MTurk by using
> `DisableHIT(hit = current_HITs)`. This will disable HITs within the
> passed vector of HIT IDs from being completed by workers, though you
> will still be charged for work which was completed before the manual
> expiration. You should refer to that function’s documentation for
> additional details.

### Retrieving HITs from MTurk

Once your HITs have been uploaded for completion, the next step is to
download the completed work. This is done using the getResults function,
which provides live status updates on completed work, downloads results,
and formats the results in a convenient dataframe. Its
current\_batch\_id argument takes a numeric integer. If running multiple
rounds of pairwise comparisons, you can change the batch id each time to
distinguish results across batches. The current\_hit\_ids argument takes
the vector of HIT IDs returned from sendComparisons, and is used to
query MTurk for the desired results. The current\_document\_ids argument
takes a matrix with the same dimensions as the object supplied to the
comparisons\_to\_send argument in sendComparisons. Though you may
directly pass that object here, it is recommended that you use a matrix
of corresponding document identifiers instead of the full document text
(note the distinction between the CompareDocs and Comparisons objects
created in the above sendComparisons example). When retry = T, the
function will continuously run in the R console providing live updates
on the percentage of completed work until all work is complete and the
full results are downloaded. When FALSE, the function will execute once,
printing the percentage of completed results and downloading those
results. The retry\_in\_seconds argument takes a number integer that
governs how long (in seconds) the function should wait before trying to
download the results again.

``` r
Results <- getResults(current_batch_id = batch_ids[i],
                      current_hit_ids = current_HITs, 
                      current_document_ids = CompareDocs,
                      retry = T, 
                      retry_in_seconds = 60)
```

### Auditing Workers

The final step in the data labeling workflow is to audit worker quality.
The auditWorkers function uses sentimentIt::fitStan to evaluate worker
quality. The specifics of this method are available in the function’s
documentation. The auditWorkers function takes one primary argument,
current\_experiment\_results, which takes a data.frame as formatted by
the getResults function. The function prints, for each worker, an MTurk
id, a posterior mean from sentimentIt::fitStan, and the number of HITs
they completed. There is also an optional argument, reference\_results,
which takes a similarly formatted data.frame, but which contains “gold
standard” comparisons that you know are correct. Supplying this argument
will contribute to estimating worker quality, but will not include the
worker IDs for workers who are exclusively in the gold standard set in
the function’s output.

``` r
auditWorkers(current_experiment_results = Results)
```

Workers with a posterior mean less than 1.0 are likely contributing poor
quality work, though you should be wary of taking action against a
worker with a score less than 1.0 if they have completed very few HITs,
as the estimate may not be completely reliable for very small n.

Upon auditing workers, there are a variety of actions you may take
against workers, all of which can be accomplished with functions in the
pyMTurkR pacakge. In particular, you may be interested in granting a
bonus (pyMTurkR::GrantBonus()), revoking a qualification in order to
prevent additional work on these tasks
(pyMTurkR::RevokeQualification()), blocking a worker from completing any
work for you in the future (pyMTurkR::BlockWorker()), or rejecting
assignment (pyMTurkR::RejectAssignments()). Each of these functions is
straightforward to run, and interested readers are advised to read the
corresponding package documentation.

> ***A note on rejecting HITs*** Even when warning workers in your
> qualification test and HIT instructions that poor quality work may be
> rejected, workers often become hostile when even a small percentage of
> their work is rejected. Some of the in-built MTurk qualifications use
> the percentage of completed HITs or the percentage of accepted HITs to
> screen workers, hence workers perceive rejection as being more costly
> to their long-term earning potential than just the foregone earning of
> your HITs alone. My recommendation — to prevent you from receiving any
> of the various complaints, threats, or condemnations that I have
> received for rejecting literal pennies worth of work — is to run many
> batches of small numbers of HITs (say, 100) and to audit worker
> quality after each batch. Instead of rejecting bad work, merely revoke
> your custom qualification from bad workers. As long as you set the
> qualification test to prevent retakes, this will prevent bad workers
> from affecting your results as soon as your diagnostics flag them as
> providing low quality work. Whatever fractions of a dollar they earned
> by spamming your HITs is simply the price you pay for a good night’s
> sleep.