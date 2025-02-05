Film Networks
================

#### Load Libraries, function scripts and data

``` r
library(tidyverse) 
library(quanteda) #for text cleaning
library(igraph) #for creating graphs
library(visNetwork) #for visualizing graphs

source("calculatecoocstats.R") #calculate co-occurrence statistics
source("grapher.R") #create graph
source("graphervf.R") #grapher 2
source("grapherdemo.R") #other grapher
source("token_filter.R") #filter tokens
load("token.all.RData")
```

### The dataset

First I retrieved movie plots from Wikipedia using the rvest package.

``` r
library(rvest)
```

#### Function to retrieve movie plots from Wikipedia

I wrote a function to retrieve ‘n’ number of plots for each year across
a specified decade. I made the function return a string of plots for the
entire decade.

``` r
plot_scraper <- function(decade = 1940, n = 200){ #declare function
  s <- character() #initialize string to store plots
  for(j in 0 : 9){ #for loop to run for 10 years in decade
    year_full = decade + j #year initialized to decade start in main script
    print(year_full)
    s_ind <- character()
    
    #create url for scraping - attach year to Wikipedia URL
    url_start <- "https://en.wikipedia.org"
    url_mid <- "/wiki/List_of_American_films_of_"
    year_start = substr(year_full, 1, 2) #get first two numbers of year
    year_end = substr(year_full, 3, 3) #get third number
    url_end <- paste(year_start, paste(year_end, j, sep = ""), sep = "") #paste them together along with j
    url <- paste(url_start, url_mid, url_end, sep = "") #paste everything together for complete URL
    
    #extract plot info
    page <- read_html(url) #read html from url
    #access the links for different movies in that particular year
    links <- html_nodes(page, "table.wikitable td i a")
    #extract hyperlinks to movie main page
    links.href <- html_attr(links, "href")
    #create accessible URLs to each of the films
    plot.links <- paste("https://en.wikipedia.org", links.href, sep = "") 
    
    #detect and delete dead/red links and buggy links (important)
    plot.links <- plot.links[!plot.links %>% str_detect("redlink", negate = FALSE)]
    plot.links <- plot.links[!plot.links %>% str_detect("Fly_by_Night", negate = FALSE)]
    plot.links <- plot.links[!plot.links %>% str_detect("Monolith", negate = FALSE)]
    plot.links <- plot.links[!plot.links %>% str_detect("The_Cruel_Path", negate = FALSE)]
    
    #take a random sample of size n from obtained links
    if(length(plot.links) > n){plot.links = sample(plot.links, n)} 
    #initialize string to hold plots
    plot <- character()
    #extract plots from each individual Wikipedia page for the movies
    for(i in 1 : n){
      #take only things under the plot heading
      plot[i] <- plot.links[i]
      if(!is.na(plot[i])){
        plot[i] <- plot.links[i] %>% read_html() %>%
          html_nodes(xpath = '//p[preceding::h2[1][contains(.,"Plot")]]') %>%
          html_text() %>%  paste(collapse = "\n")
        }
      else{
        plot[i] <- ""
      }
      print(i)
    }
    #remove plots with no info - some movies don't have a plot section
    #mostly the less popular ones
    s_ind <- plot[plot != ""] #store all non-empty plots for the year into a new variable
    s <- c(s, s_ind) #bind the string for individual year with the string for the entire decade
  }
  return(s)
}
```

#### Function to tokenize extracted plots

I also wrote a function to tokenise a given string with information
about the part of speech of each word using the spacy library.
Additionally I also used the genderizeR package to classify character
names to their genders. Using this information, I converted all male
character names to one single category and all female character names to
a single category.

``` r
#connect your python to the environment where you installed spacy
reticulate::use_virtualenv("~/spacynlp-env", required = TRUE)   
reticulate::py_config() #check whether configuration is right

#load libraries
library(spacyr) #for NLP
spacy_initialize(model = "en_core_web_sm") #spacy language model
library(quanteda) #for text cleaning
library(genderizeR) #for assigning gender


film_tokenizer <- function(plot_string, metadata){ #declare function
  
  #convert string into text corpus for cleaning
  print("converting into corpus") #print status
  s <- corpus(pllot_strings, docvars = metadata) #make a corpus object and attach doc variables (year info)
  s <- corpus_reshape(s, to = "sentences") #split it into sentences
  docvars_complete <- docvars(s)   #count number of sentences - useful later when assigning docvars to spacy object
  
  #parse it into tokens using spacy - this is where the magic happens
  print("starting parse using spacy") #print status
  toks.spacy <- spacy_parse(s) %>%
    entity_consolidate() %>% #this combines single entities into one unit
    #by replacing ' ' with '_' e.g. John Locke becomes John_Locke
    as.tokens(include_pos = "pos") #include parts of speech information
  
  #extract entities person entities from the text i.e. movie characters
  print("tokens intitial parse") #print status
  ents.spacy <- spacy_parse(s, entity = TRUE) %>% 
    entity_extract(concatenator = "_") %>% #extract entities
    filter(entity_type == "PERSON") %>% #filter persons
    distinct(entity) #find distrinct persons i.e. remove duplicates
  
  #extract gender information
  print("starting gender extract") #print status
  ents.spacy.temp <- ents.spacy %>% mutate(entity = str_replace_all(entity, "[_]", " ")) #replace '_' with ' ' for gender extraction
  print("finding given names") #print status
  #find given names
  givenNames = findGivenNames(ents.spacy.temp$entity, progress = FALSE, apikey = 
  "31d8c048c93f385ba2de144836d8d0f5") #identify given names
  #find gender from given names
  gender_data = genderize(ents.spacy.temp$entity, genderDB = givenNames, progress = FALSE)
  #store all entities
  entity.all <- gender_data %>% mutate(name = ents.spacy$entity) %>% select(name, gender) #keep only required columns
  
  #filter gender entities
  print("gender extraction complete") #print status
  entity.male <- entity.all %>% filter(gender == "male") #filter male entities
  entity.female <- entity.all %>% filter(gender == "female") #filter female entities
  n.males <- nrow(entity.male) #count male entites
  n.female <- nrow(entity.female) #count female entities
  #create string to use for replacing individual character names to general term
  m.repl <- rep("Male/CHARACTERS", n.males) #males
  f.repl <- rep("Female/CHARACTERS", n.female) #females
 
  #create final tokens
   print("creating tokens") #print status
    toks.all <- toks.spacy %>% 
    tokens_select(pattern = c("*/NOUN", "*/VERB", "*/ENTITY", "*/ADJ")) %>% #select only nouns, verbs, adjectives, entities
    tokens_replace(pattern = paste(entity.male$name, "/ENTITY", sep = ""), replacement = m.repl) %>% #replce ind. characters with genereal term - male
    tokens_replace(pattern = paste(entity.female$name, "/ENTITY", sep = ""), replacement = f.repl) %>% #replce ind. characters with genereal term - female
    tokens_remove(c("", "'s", "-", "ex", "-/NOUN", "*/ENTITY", "-/ADJ", "-/VERB")) #remove buggy tokens
  
  print("done") #print status
  toks.all$decade = docvars_complete #assign decade to tokens
  return(toks.all)
}
```

#### Using the functions to create dataset for analysis

I created the dataset for my analysis using the two functions above.
First I used the plot scraper function to extract movie plots from all
decades.

``` r
#scrape all plots across all decades
s_all.i <- character() #declare string to hold all plot info
year_plots <- data.frame() #declare data frame to hold info on number of plots scraped per year

for(i in seq(from=1940, to=2010, by=10)){ #run for loop to get each decade from 1940 to 2010
  print(paste(i, "start", sep = " - ")) #print status
  s_this.i <- plot_scraper(i, 200) #plot 200 movies from every year, across the entire decade i
  s_all.i <- c(s_all.i, s_this.i) #merge with string for overall data
  n_plots = length(s_this.i) #find number of plots in particular decade
  year_plots.temp <- data.frame(year = as.character(i), times = n_plots) #organise
  year_plots <- rbind(year_plots, year_plots.temp) #bind to overall data for plots/decade info
}
s_docvars <- rep(year_plots.final$year, times = year_plots.final$times) #create docvars to use for tokenizing (year info for each plot)
```

After extracting all the movie plots, I used the film tokenizer function
to tokenize the movie plots.

``` r
#tokenise data
token.all <- film_tokenizer(s_all.i, s_docvars)
```

A look at a sample of the final dataset

``` r
head(token.all, 5)
```

    ## Tokens consisting of 5 documents and 1 docvar.
    ## text1.1 :
    ## [1] "father/NOUN"     "Male/CHARACTERS" "Male/CHARACTERS" "joins/VERB"     
    ## [5] "promised/VERB"   "land/NOUN"      
    ## 
    ## text1.2 :
    ##  [1] "Male/CHARACTERS" "learns/VERB"     "father/NOUN"     "died/VERB"      
    ##  [5] "military/ADJ"    "expedition/NOUN" "consoled/VERB"   "schoolmate/NOUN"
    ##  [9] "friend/NOUN"     "Male/CHARACTERS" "Male/CHARACTERS"
    ## 
    ## text1.3 :
    ## [1] "Male/CHARACTERS"   "adult/NOUN"        "accomplished/ADJ" 
    ## [4] "backwoodsman/NOUN" "sells/VERB"        "family/NOUN"      
    ## [7] "farm/NOUN"         "order/NOUN"        "settle/VERB"      
    ## 
    ## text1.4 :
    ##  [1] "saying/VERB"     "Male/CHARACTERS" "played/VERB"     "adult/NOUN"     
    ##  [5] "Male/CHARACTERS" "Male/CHARACTERS" "tricked/VERB"    "meeting/VERB"   
    ##  [9] "several/ADJ"     "members/NOUN"    "high/ADJ"        "society/NOUN"   
    ## [ ... and 10 more ]
    ## 
    ## text1.5 :
    ## [1] "snub/VERB"       "discover/VERB"   "common/ADJ"      "farmer/NOUN"    
    ## [5] "landed/ADJ"      "gentlemen/NOUN"  "Male/CHARACTERS" "implied/VERB"

``` r
#convert tokens to all lower
token.all <- tokens_tolower(token.all) #convert all tokens to lower
token.all = token.all %>% tokens_remove(c('ex/adj', 'ex/noun'))
```

#### Exploratory analysis

Since the data obtained is unequal for each decade due to some inherent
factors, I wanted to check how the certain parameters are distributed
acrosst he decades.

``` r
#plot the number of movies in each decade
sents_df = data.frame(decade = as.character(), 
                       n_sents = as.numeric())
for(i in seq(1940, 2010, 10)){
  n_sents = ndoc(tokens_subset(token.all, decade == i))
  sents_t = data.frame(decade = as.character(i), 
                       n_sents = as.numeric(n_sents))
  sents_df = rbind(sents_df, sents_t)
}

ggplot(sents_df, aes(x = decade, n_sents)) +
  geom_bar(stat = 'identity', width = 0.5, color = 'black',
           position = position_dodge(width = 0.4)) +
  theme_linedraw() + ylab('no. of sentences')
```

![](README_files/figure-gfm/unnamed-chunk-9-1.jpeg)<!-- -->

``` r
#plot number of words per sentence across decades
words_df = data.frame(decade = as.character(), 
                       n_words = as.numeric())
for(i in seq(1940, 2010, 10)){
  n_words = sum(ntoken(tokens_subset(token.all, decade == i)))
  words_t = data.frame(decade = as.character(i), 
                       n_words = as.numeric(n_words))
  words_df = rbind(words_df, words_t)
}

words_df$wordspsents = words_df$n_words/sents_df$n_sents

ggplot(words_df, aes(x = decade, wordspsents)) +
  geom_bar(stat = 'identity', width = 0.5, color = 'black',
           position = position_dodge(width = 0.4)) +
  theme_linedraw() + ylab('words per sentence')
```

![](README_files/figure-gfm/unnamed-chunk-9-2.jpeg)<!-- -->

``` r
set.seed(42) #for replication
#UPDATE - general version
#sample based on min in a decade
token.all = tokens_sample(token.all, size = 22638, replace = FALSE, prob = NULL, by = decade)
```

### Graph Construction

#### Function to create co-occurence network using a given set of tokenised text

``` r
grapherdemo <- function(numberOfCoocs, toks, measure = "LOGLIK"){
  #oppositeg = ifelse(coocTerm == 'male/characters', 'female/characters', 'male/characters')
  #coocTerm = 'male/characters'
  #### graph df function
  graph_df <- function(coocTerm){
  minimumFrequency = 10
  binDTM <- toks %>% 
    dfm() %>% 
    dfm_trim(min_docfreq = minimumFrequency) %>% 
    dfm_weight("boolean")
  
  coocs <- calculateCoocStatistics(coocTerm, binDTM, measure)

  # Display the numberOfCoocs main terms
  imm.coocs <- names(coocs[1:numberOfCoocs])
  logs <- coocs[1:numberOfCoocs]
  
  resultGraph <- data.frame(from = character(), to = character(), sig = numeric(0))
  
  # The structure of the temporary graph object is equal to that of the resultGraph
  tmpGraph <- data.frame(from = character(), to = character(), sig = numeric(0))
  
  # Fill the data.frame to produce the correct number of lines
  tmpGraph[1:numberOfCoocs, 3] <- coocs[1:numberOfCoocs]
  # Entry of the search word into the first column in all lines
  tmpGraph[, 1] <- coocTerm
  # Entry of the co-occurrences into the second column of the respective line
  tmpGraph[, 2] <- names(coocs)[1:numberOfCoocs]
  # Set the significances
  tmpGraph[, 3] <- coocs[1:numberOfCoocs]
  
  # Attach the triples to resultGraph
  resultGraph <- rbind(resultGraph, tmpGraph)
  
  # Iteration over the most significant numberOfCoocs co-occurrences of the search term
  for (i in 1:numberOfCoocs){

    # Calling up the co-occurrence calculation for term i from the search words co-occurrences
    newCoocTerm <- names(coocs)[i]
    coocs2 <- calculateCoocStatistics(newCoocTerm, binDTM, measure="LOGLIK")

    #print the co-occurrences
    coocs2[1:10]

    # Structure of the temporary graph object
    tmpGraph <- data.frame(from = character(), to = character(), sig = numeric(0))
    tmpGraph[1:numberOfCoocs, 3] <- coocs2[1:numberOfCoocs]
    tmpGraph[, 1] <- newCoocTerm
    tmpGraph[, 2] <- names(coocs2)[1:numberOfCoocs]
    tmpGraph[, 3] <- coocs2[1:numberOfCoocs]

    #Append the result to the result graph
    resultGraph <- rbind(resultGraph, tmpGraph[2:length(tmpGraph[, 1]), ])
  }

  # Sample of some examples from resultGraph
  #resultGraph[sample(nrow(resultGraph), 6), ]
  list_graph = list()
  list_graph[[1]] = imm.coocs
  list_graph[[2]] = resultGraph
  list_graph[[3]] = logs
  return(list_graph)
  }
  male_list = graph_df('male/characters')
  female_list = graph_df('female/characters')
  male = male_list[[2]]
  male_coocs = male_list[[1]]
  male_logs = male_list[[3]]
  female = female_list[[2]]
  female_coocs = female_list[[1]]
  female_logs = female_list[[3]]
  complete = rbind(male, female)
  tail(complete)
  complete <- distinct(complete)
  nrow(complete)
  # set seed for graph plot
  set.seed(42)
  resultGraph = complete
  names(complete)[3] = 'weight'
  head(complete)
  # Create the graph object as undirected graph
  graphNetwork <- graph.data.frame(resultGraph, directed = F)
  E(graphNetwork)$weight = complete$weight
  is_weighted(graphNetwork)
  
  
  # Identification of all nodes with less than 2 edges
  verticesToRemove <- V(graphNetwork)[degree(graphNetwork) < 2]
  # These edges are removed from the graph
  #graphNetwork <- delete.vertices(graphNetwork, verticesToRemove) 
  #imm.coocs
  
  #for vertices #####
  #male to female - not needed
  #ftm = rowSums(ends(graphNetwork, es = E(graphNetwork), names = T) == c('female/characters', 'male/characters'))
  #female primary nodes
  fto = ends(graphNetwork, es = E(graphNetwork), names = T)[,1] == 'female/characters'
  fto2 = ends(graphNetwork, es = E(graphNetwork), names = T)[,2] == 'female/characters'
  #male primary nodes
  mto = ends(graphNetwork, es = E(graphNetwork), names = T)[,1] == 'male/characters'
  mto2 = ends(graphNetwork, es = E(graphNetwork), names = T)[,2] == 'male/characters'
  
   #female connections
  fc = ends(graphNetwork, es = E(graphNetwork), names = T)[,2][as.logical(fto)]
  fc2 = ends(graphNetwork, es = E(graphNetwork), names = T)[,1][as.logical(fto2)]
  #male connections
  mc = ends(graphNetwork, es = E(graphNetwork), names = T)[,2][as.logical(mto)]
  mc2 = ends(graphNetwork, es = E(graphNetwork), names = T)[,1][as.logical(mto2)]
  
  main_cm = c(mc, mc2)
  main_cf = c(fc, fc2)
  maf = intersect(main_cm, main_cf)
  intersect = intersect(male_coocs, female_coocs)
  
  # Assign colors to nodes (search term blue, primary green, others orange)
  V(graphNetwork)$color <- ifelse(V(graphNetwork)$name == c('male/characters'), adjustcolor('cornflowerblue', alpha = 0.9),
                                  ifelse(V(graphNetwork)$name %in% c('female/characters'), adjustcolor('orange', alpha = 0.9),
                                         ifelse(V(graphNetwork)$name %in% c(intersect), adjustcolor('purple', alpha = 0.8),
                                  ifelse(V(graphNetwork)$name %in% male_coocs, adjustcolor('cornflowerblue', alpha = 0.8),
                                         ifelse(V(graphNetwork)$name %in% female_coocs, adjustcolor('orange', alpha = 0.9), adjustcolor('grey', alpha = 0.4))))))
  
  #V(graphNetwork)$color <- ifelse(V(graphNetwork)$name %in% fc, 'orange', V(graphNetwork)$color)
  # Set edge colors
  #E(graphNetwork)$color <- adjustcolor("DarkGray", alpha.f = .5)
  # scale significance between 1 and 10 for edge width
  E(graphNetwork)$width <- scales::rescale(E(graphNetwork)$sig, to = c(1, 10))
  
  E(graphNetwork)$color <- adjustcolor('DarkGray', alpha.f = 0.4)
   

  # Set edges with radius
  E(graphNetwork)$curved <- 0.15 
  # Size the nodes by their degree of networking (scaled between 5 and 15)
  V(graphNetwork)$size <- scales::rescale(degree(graphNetwork), to = c(10, 25))
  
  # Define the frame and spacing for the plot
  par(mai=c(0,0,1,0)) 
  
  graph_list <- list()
  graph_list[[1]] <- graphNetwork #network object
  graph_list[[2]] <- female_logs #names of co-occs (redundant)
  graph_list[[3]] <- male_logs #data frame of co-occs and significance
  return(graph_list)
}
```

``` r
#add shiny toggle secondary, shiny toggle nodes
graph_demo = grapherdemo(5, token_filter3('all', 1940, 2020, token.all)) 
```

    ## Loading required package: Matrix

    ## 
    ## Attaching package: 'Matrix'

    ## The following objects are masked from 'package:tidyr':
    ## 
    ##     expand, pack, unpack

``` r
g_demo = graph_demo[[1]]
visIgraph(g_demo)
```

![](README_files/figure-gfm/unnamed-chunk-12-1.jpeg)<!-- -->

### Complete Graph

``` r
graph = grapherdemo(21, token_filter3('all', 1940, 2020, token.all)) #create graph
 female_primary = graph[[2]] #20 female primary nodes
 male_primary = graph[[3]] #20 male primary nodes
 g = graph[[1]] #save graph as g
 visIgraph(g) #%>% visNodes(font = list(size = 26))  #display
```

![](README_files/figure-gfm/unnamed-chunk-13-1.jpeg)<!-- -->

### Significant Tropes in the Network

``` r
#Top secondary co-occurences
 #male
 dmat = distances(graph[[1]], v=V(graph[[1]]), to='male/characters') #compute path weights
 male_c = dmat[, 'male/characters'] #secondary to male
 male_c = sort(male_c, decreasing = T)[1:20] #sort top 20
 
 #female
 fmat = distances(graph[[1]], v=V(graph[[1]]), to='female/characters') #compute path weights
 female_c = fmat[, 'female/characters'] #secondary to male
 female_c = sort(female_c, decreasing = T)[1:20] #sort top 20
 
 #store all secondary
 allc = c(male_c, female_c) 
 allc = sort(allc, decreasing = T) #sort decreasing

 #edge colors
 all_edges = ends(g, es = E(g), names = T) #store all edges
 all_edges = as.data.frame(all_edges) #convert to dataframe
 
 #check 
 all_edges$V2[all_edges$V1 == 'female/characters']
```

    ##  [1] "daughter/noun"     "sister/noun"       "love/noun"        
    ##  [4] "mother/noun"       "husband/noun"      "relationship/noun"
    ##  [7] "affair/noun"       "house/noun"        "marriage/noun"    
    ## [10] "girl/noun"         "marry/verb"        "woman/noun"       
    ## [13] "pregnant/adj"      "wedding/noun"      "married/verb"     
    ## [16] "love/noun"         "mother/noun"       "relationship/noun"
    ## [19] "affair/noun"       "marriage/noun"     "girl/noun"        
    ## [22] "woman/noun"        "pregnant/adj"      "wedding/noun"

``` r
 #male_c = male_c[names(male_c != 'beach/noun')]
 male.sec_bool <- all_edges$V2 %in% names(male_c)  #create bool of all male secondary co-oocs
 female.sec_bool <- all_edges$V2 %in% names(female_c)  #create bool of all female secondary co-oocs
 
 edge.start <- ends(g, es = E(g), names = F)[,1]
 # E(g)$color <-  ifelse(male.sec_bool == TRUE, V(g)$color[edge.start], 
 #                       ifelse(female.sec_bool == TRUE, V(g)$color[edge.start],
 #                       adjustcolor('grey', alpha=0.4)))
 
 male_ps = intersect(all_edges$V1[male.sec_bool], names(male_primary))
 female_ps = intersect(all_edges$V1[female.sec_bool], names(female_primary))
 all_edges$V1[female.sec_bool]
```

    ##  [1] "friend/noun"       "friend/noun"       "is/verb"          
    ##  [4] "is/verb"           "is/verb"           "kill/verb"        
    ##  [7] "takes/verb"        "love/noun"         "love/noun"        
    ## [10] "love/noun"         "love/noun"         "love/noun"        
    ## [13] "love/noun"         "love/noun"         "relationship/noun"
    ## [16] "relationship/noun" "relationship/noun" "wedding/noun"     
    ## [19] "wedding/noun"      "wedding/noun"

``` r
 #color only primary tropes that have a path
 #mprimary_tropes = c('is/verb', 'friend/noun', 'takes/verb', 'tells/verb',
      #               'kill/verb', 'agent/noun', 'help/noun', 
      #               'brother/noun', 'former/adj')
 mprimary_tropes = male_ps
 mprimary_tropes = mprimary_tropes[mprimary_tropes != 'female/characters']
 m_pcolor = paste('male/characters', mprimary_tropes)
 all_edges$V3 = paste(all_edges$V1, all_edges$V2)
 mp_bool = all_edges$V3 %in% m_pcolor
 
 #fprimary_tropes = c('love/noun', 'marriage/noun', 'relationship/noun',
      #               'tells/verb')
fprimary_tropes = female_ps
 f_pcolor = paste('female/characters', fprimary_tropes)
 all_edges$V3 = paste(all_edges$V1, all_edges$V2)
 all_edges$V1[all_edges$V2 == 'tells/verb']
```

    ## [1] "male/characters" "male/characters"

``` r
 fp_bool = all_edges$V3 %in% f_pcolor
 
 E(g)$color <-  adjustcolor('grey', alpha=0.9)
 
 E(g)$color <-  ifelse(mp_bool == TRUE, V(g)$color[edge.start], 
                       ifelse(fp_bool == TRUE, V(g)$color[edge.start],
                              
                              ifelse(male.sec_bool == TRUE, V(g)$color[edge.start],
                                     ifelse(female.sec_bool == TRUE, V(g)$color[edge.start],       
                                            adjustcolor('grey', alpha=0.4)))))
 
 visIgraph(g)
```

![](README_files/figure-gfm/unnamed-chunk-14-1.jpeg)<!-- -->

``` r
 #all_edges$V3[malet_bool]
 
 V(g)$color <- ifelse(V(g)$name == c('male/characters'), adjustcolor('cornflowerblue', alpha = 0.9),
                      ifelse(V(g)$name %in% c('female/characters'), adjustcolor('orange', alpha = 0.9),
                             ifelse(V(g)$name %in% c(intersect(mprimary_tropes, fprimary_tropes)), adjustcolor('purple', alpha = 0.9),
                                    ifelse(V(g)$name %in% mprimary_tropes, adjustcolor('cornflowerblue', alpha = 0.9),
                                           ifelse(V(g)$name %in% fprimary_tropes, adjustcolor('orange', alpha = 0.9),
                                                  ifelse(V(g)$name %in% c(names(male_c), names(female_c)), adjustcolor('darkgrey', alpha = 0.9),
                                                         adjustcolor('grey', alpha = 0.2)))))))
 
 #V(g)$color <- when(V(g)$name %in% 'male/character', adjustcolor('red', alpha = 0.8))
 #visIgraph(g)
 
 
 keep_nodes = names(c(allc, male_primary, female_primary))
 keep_nodes = c(keep_nodes, 'male/characters', 'female/characters')
 remove_nodes = names(V(g))[!names(V(g)) %in% keep_nodes]
 
 g_trim <- g - remove_nodes
 visIgraph(g_trim) %>% visNodes(font = list(size = 26))
```

![](README_files/figure-gfm/unnamed-chunk-14-2.jpeg)<!-- -->
