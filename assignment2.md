Data analysis assignment 2
================
Ryan Kennelly
07.02.2020

In this assignment you will work with relational data, i.e. data coming
from different data tables that you can combine using keys. Please read
ch.13 from R for Data Science before completing this assignment –
<https://r4ds.had.co.nz/relational-data.html>.

## Read data

We will work with three different tables: household roster from wave 8
(*h\_egoalt*), stable characteristics of individuals (*xwavedat*), and
household data from wave 8 (*h\_hhresp*).

``` r
library(tidyverse)
# You need to complete the paths to these files on your computer.
Egoalt8 <- read_tsv("C:/Users/User/Documents/University/Year 2/Term 2/Data 3/data/UKDA-6614-tab/tab/ukhls_w8/h_egoalt.tab")
Stable <- read_tsv("C:/Users/User/Documents/University/Year 2/Term 2/Data 3/data/UKDA-6614-tab/tab/ukhls_wx/xwavedat.tab")
Hh8 <- read_tsv("C:/Users/User/Documents/University/Year 2/Term 2/Data 3/data/UKDA-6614-tab/tab/ukhls_w8/h_hhresp.tab")
```

## Filter household roster data (10 points)

The **egoalt8** data table contains data on the kin and other
relationships between people in the same household. In each row in this
table you will have a pair of individuals in the same household: ego
(identified by *pidp*) and alter (identified by *apidp*).
*h\_relationship\_dv* shows the type of relationship between ego and
alter. You can check the codes in the Understanding Society codebooks
here –
<https://www.understandingsociety.ac.uk/documentation/mainstage/dataset-documentation>.

First we want to select only pairs of individuals who are husbands and
wives or cohabiting partners (codes 1 and 2). For convenience, we also
want to keep only the variables *pidp*, *apidp*, *h\_hidp* (household
identifier), *h\_relationship\_dv*, *h\_esex* (ego’s sex), and *h\_asex*
(alter’s sex).

``` r
Partners8 <- Egoalt8 %>%
        filter(h_relationship_dv > 0 & h_relationship_dv < 3)  %>%
        select(pidp, apidp, h_hidp, h_relationship_dv, h_sex, h_asex)
table(Partners8$h_relationship_dv)
```

    ## 
    ##     1     2 
    ## 22270  5160

Each couple now appears in the data twice: 1) with one partner as ego
and the other as alter, 2) the other way round. Now we will only focus
on heterosexual couples, and keep one observation per couple with women
as egos and men as their alters.

``` r
Hetero8 <- Partners8 %>%
        # filter out same-sex couples
        filter(h_sex != h_asex) %>%
        # keep only one observation per couple with women as egos
        filter(h_asex == 1)
table(Hetero8$h_sex)
```

    ## 
    ##     2 
    ## 13615

``` r
table(Hetero8$h_asex)
```

    ## 
    ##     1 
    ## 13615

## Recode data on ethnicity (10 points)

In this assignment we will explore ethnic endogamy, i.e. marriages and
partnerships within the same ethnic group. First, let us a create a
version of the table with stable individual characteristics with two
variables only: *pidp* and *racel\_dv* (ethnicity).

``` r
Stable2 <- Stable %>%
        select(pidp, racel_dv)
```

Let’s code missing values on ethnicity (-9) as NA.

``` r
Stable2 <- Stable2 %>%
        mutate(racel_dv = recode(racel_dv, `-9` = NA_real_))
```

Now let us recode the variable on ethnicity into a new binary variable
with the following values: “White” (codes 1 to 4) and “non-White” (all
other codes).

``` r
Stable2 <- Stable2 %>%
        mutate(race = case_when(
                between(racel_dv, 1, 4) ~ "White",
                racel_dv > 4 ~ "non-White"
        ))
table(Stable2$race)
```

    ## 
    ## non-White     White 
    ##     18121     81327

## Join data (30 points)

Now we want to join data from the household roster (*Hetero8*) and the
data table with ethnicity (*Stable2*). First let us merge in the data on
ego’s ethnicity. We want to keep all the observations we have in
*Hetero8*, but we don’t want to add any other individuals from
*Stable2*.

``` r
JoinedEthn <- Hetero8 %>%
        left_join(Stable2, by = "pidp")
```

Let us rename the variables for ethnicity to clearly indicate that they
refer to egos.

``` r
JoinedEthn <- JoinedEthn %>%
        rename(egoRacel_dv = racel_dv) %>%
        rename(egoRace = race)
```

Now let us merge in the data on alter’s ethnicity. Note that in this
case the key variables have different names in two data tables; please
refer to the documentation for your join function (or the relevant
section from R for Data Science) to check the solution for this problem.

``` r
JoinedEthn <- JoinedEthn %>%
        left_join(Stable2, by = c("apidp" = "pidp")) 
```

Renaming the variables for alters.

``` r
JoinedEthn <- JoinedEthn %>%
        rename(alterRacel_dv = racel_dv) %>%
        rename(alterRace = race)
```

## Explore probabilities of racial endogamy (20 points)

Let us start by looking at the joint distribution of race (White
vs. non-White) of both partners.

``` r
TableRace <- JoinedEthn %>%
        # filter out observations with missing data
        filter(egoRace != "NA_character_" & alterRace != "NA_character_") %>%
        count(egoRace, alterRace)
TableRace
```

    ## # A tibble: 4 x 3
    ##   egoRace   alterRace     n
    ##   <chr>     <chr>     <int>
    ## 1 non-White non-White  1790
    ## 2 non-White White       326
    ## 3 White     non-White   266
    ## 4 White     White      9694

Now calculate the following probabilities: 1) for a White woman to have
a White partner, 2) for a White woman to have a non-White partner, 3)
for a non-White woman to have a White partner, 4) for a non-White woman
to have a non-White partner.

Of course, you can simply calculate these numbers manually. However, the
code will not be reproducible: if the data change the code will need to
be changed, too. Your task is to write reproducible code producing a
table with the required four probabilities.

``` r
TableRace %>%
        # group by ego's race to calculate sums
        group_by(egoRace) %>%
        # create a new variable with the total number of women by race
        mutate(wn = sum(n)
        ) %>%
        # create a new variable with the required probabilities 
        mutate(perc = n / sum(n) * 100)
```

    ## # A tibble: 4 x 5
    ## # Groups:   egoRace [2]
    ##   egoRace   alterRace     n    wn  perc
    ##   <chr>     <chr>     <int> <int> <dbl>
    ## 1 non-White non-White  1790  2116 84.6 
    ## 2 non-White White       326  2116 15.4 
    ## 3 White     non-White   266  9960  2.67
    ## 4 White     White      9694  9960 97.3

## Join with household data and calculate mean and median number of children by ethnic group (30 points)

1)  Join the individual-level file with the household-level data from
    wave 8 (specifically, we want the variable for the number of
    children in the household).
2)  Select only couples that are ethnically endogamous (i.e. partners
    come from the same ethnic group) for the following groups: White
    British, Indian, and Pakistani.
3)  Produce a table showing the mean and median number of children in
    these households by ethnic group (make sure the table has meaningful
    labels for ethnic groups, not just numerical codes).
4)  Write a short interpretation of your results. What could affect your
    findings?

<!-- end list -->

``` r
#1
Joined1 <- JoinedEthn %>%
        full_join(Hh8, by = "h_hidp") %>%
        select(pidp, apidp, h_hidp, h_relationship_dv, h_sex, h_asex, egoRacel_dv, egoRace, alterRacel_dv, alterRace, h_nkids_dv)
#2
EndoEthno <- Joined1 %>%
        filter(egoRacel_dv == 1 & alterRacel_dv == 1 | egoRacel_dv == 9 & alterRacel_dv == 9 | egoRacel_dv == 10 & 
                    alterRacel_dv == 10) %>%
        mutate(CouplesRace = case_when(
                egoRacel_dv == 1 & alterRacel_dv == 1 ~ "White British",
                egoRacel_dv == 9 & alterRacel_dv == 9 ~ "Indian",
                egoRacel_dv == 10 & alterRacel_dv == 10 ~ "Pakistani"
        ))
#3
Aggr1 <- EndoEthno %>%
  select(CouplesRace, h_nkids_dv) %>%
        group_by(CouplesRace) %>%
  summarise(
    MeanofChildren = mean(h_nkids_dv, na.rm = TRUE),
    MedianofChildren = median(h_nkids_dv, na.rm = TRUE)
  )
  Aggr1
```

    ## # A tibble: 3 x 3
    ##   CouplesRace   MeanofChildren MedianofChildren
    ##   <chr>                  <dbl>            <dbl>
    ## 1 Indian                 0.955                1
    ## 2 Pakistani              1.81                 2
    ## 3 White British          0.565                0

Findings: when observing white British respondents in table Aggr1 it can
be observed that the mean number of children per couple is less than 1
which is impossible in this case, this is due to a large portion of the
respondants having 0 children which is also reinforced by the median
displaying 0 thus skewing the data. From this it can be said that White
British couples are far less likely to have any children than Indian or
Pakistani who show means at 1 (2dp) and 1.81 respectivly. The Pakistani
respondents are most likely to have over one child per household whith
the mean displaying 1.81(2dp) and 2 in the median. Indian respondents
are most likely to have 1 child per household. However the lage number
of responses that = 0 may have effected all three race groupings so it
would be worth checking the number of 0’s per race and posibly removing
them and comparing both tables.
