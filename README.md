## BMB faculty publications

Please know that this code is not perfect, and you still have to do something manually. For example, some names have special coding e.g. Farr&#xe9 or Ru&#xdf. Also, some journal use all capital letters for artical names. Please fix those before publishing the resarch results.

The [faculty list](https://github.com/hsinyenwu/BMB_faculty_publications/blob/main/faculty%20list_2022.12_AI.xlsx) is scraped from BMB regular faculty website with ChatGPT.

### Code
First load the library and the faculty names:
```
#rm(list=ls()) 
library(easyPubMed)
library(dplyr)
library(kableExtra)
library(lubridate)
library(stringr)
library(openxlsx)
FList <- read.xlsx("~/Desktop/faculty list_2022.12_AI.xlsx")
```

### Next you need to provide parameters:
I will suggest to use days from **Friday of previous week to Thursday of this week**, so we will not miss any publications. This is because BMB weekly is published every Friday.

```
START <- "2022-12-09"
END <- "2022-12-15"
YEAR <- "2022"
```

Loop for PubMed searching:
```
#Empty the vector
PAPERS <- c()

#Loop for all faculty names.
suppressMessages(for(i in 1:length(FList$FName)){
  # print(FList$FName[i])
  #Extract
  dami_on_pubmed <- get_pubmed_ids(paste0(FList$FName[i],"[AU] AND Michigan State University [AD]", " AND ", YEAR, "[PDAT]"))
  if(dami_on_pubmed$Count!=0){
    # Fetch the data
    my_abstracts_xml <- fetch_pubmed_data(dami_on_pubmed)
    # Store Pubmed Records as elements of a list
    all_xml <- articles_to_list(my_abstracts_xml)
    # print(all_xml)
    # Perform operation (use lapply here, no further parameters)
    final_df <- do.call(rbind, lapply(all_xml, article_to_df, max_chars = -1, getAuthors = TRUE))
    # remove HTML tages (some titles have HTML tages)
    final_df$title <- gsub("<.*?>", "", final_df$title)
    final_df$midname <- ifelse(grepl(pattern = " ",final_df$firstname),paste0(gsub(".*\\s", "", paste0(final_df$firstname)),"."),"")
    final_df$firstinitial <- paste0(substr(final_df$firstname,1,1),".")
    final_df$author <- paste(final_df$lastname,paste0(final_df$firstinitial,final_df$midname),sep=", ")
    # final_df
    # Some names like: Farr&#xe9 or Ru&#xdf... need to be fixed
    
    final_df <- final_df %>% dplyr::select(author, title, jabbrv, doi, pmid, year, month, day)
    final_df1 <- final_df %>%
      group_by(title, year, month, day, jabbrv, doi) %>%
      summarize(authors = str_c(author, collapse = ", ")) %>% as.data.frame()
    
    # Select fields
    final_df2 <- final_df1 %>% dplyr::select(authors, title, jabbrv, year, month, day, doi)
    # create a date variable
    final_df2$date <- ymd(paste(final_df2$year,final_df2$month,final_df2$day,sep="-"))
    final_df2$date2 <- format(final_df2$date, format="%B %Y")
    # Filter for a specific week with the date variable
    
    final_df2 <- final_df2 %>% filter(date >= as.Date(START), date <= as.Date(END))
    
    if (nrow(final_df2)>0) {
      print(final_df2)
      final_df3 <- final_df2 %>% dplyr::select(authors,title,jabbrv,date2)
      final_df3$combine <- paste0(final_df3$authors," \"",
                                  final_df3$title,"\". ",
                                  final_df3$jabbrv,". ",
                                  final_df3$date2)
      PAPERS <- c(PAPERS,final_df3$combine)
    } else {next}
  }else {next}
})

df <- data.frame(PAPERS=PAPERS) 
df
```
Save the file:
```
write.xlsx(df,"~/Desktop/12_09_12_15_2022_papers.xlsx",)
```









