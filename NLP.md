---

title:        NLP - Konzept und Idee 
author:       Yohan Park
tags:         NLP, Textmining

---

# NLP - Konzept und Idee:

- [Datensicht](#datensicht)
  - [Gegenstand der Daten](#gegenstand-der-daten)
  - [Fragestellung](#fragestellung)
  - [Weitere Überlegungen](#weitere-überlegungen)
  
- [Methodische Sicht](#methodische-sicht)
  - [Named Entity Recognition](#named-entity-recognition)
  - [Text Klassifikation](#text-klassifikation)
  
## Datensicht
### Gegenstand der Daten

* Historische Verfassungstexte der Kolonien im ehemaligen britischen Empire bzw. Mitgliedstaaten des Commonwealth.
* Dateiformat: Plain text

### Fragestellung 

* Inwieferen unterscheiden sich die Gesetztexte voneinander? 
  * mögliche Adaption der britischen Verfassung ? 
  * (kulturelle) Einflüsse auf die ehm. Kolonialländer?
  
### Weitere Überlegungen

* Aufbereitung der weiteren Daten 
  * Beispiel Hongkong 
  * [Hong Kong Royal Instructions 1917][^1]
  
  
* Eventuelle Kategorisierung der Daten nach Kontinent? 
  * Führt zur Verbesserung der Übersichtlichkeit der Daten

## Methodische Sicht
  
### Named Entity Recognition
  
* Automatische Erkennung der klar benennbaren Elementen: 
  * Personen (Angela Merkel, Donald Trump)
  * Orte (Mainz, Darmstadt)
  * Organisationen (UNO, UNESCO)
  * ETC... 
  
  ![NER](./Image/NER.png)
  
* Library für NER: 
  * Stanford Named Entity Tagger
  * Spacy [^2]
  * Blackstone (Spacy-Modell)

* HowTo:

  * Plaintext-Dateien importieren  
  * Umwandlung des Dateiformats in Tidytext-Format 
    * Tidytext: Dataframe, in dem pro Zeile nur ein Token (z.B. Wort) steht.
  * Annotation der Entitäten durch SpaCy-Parser
  * Extraktion der Entitäten
  ```
  mycorpus_tidy <-tidy(mycorpus) 

  corpus_spacyr <- mycorpus_tidy %>% 
    unnest_tokens(word, text) %>% 
    anti_join(stop_words) %>% 
    count(title, word, sort=TRUE)%>%
    mutate(ner = map(word, ~spacy_parse(., entity = TRUE)))
  
  entity_df <- corpus_spacyr %>% 
    select(word, ner) %>% 
    mutate(entity = map(ner, ~ count(., entity, sort = TRUE) )) %>% 
    unnest(entity)
  ```
  * Zusammenfassung der erkannten Entitäten
  ```
  entity_df %>% 
    group_by(entity) %>% 
    summarise(entity_sum = sum(n)) %>% 
    arrange(desc(entity_sum)) %>% 
   filter(entity != "")
   
  # A tibble: 8 x 2
    entity     entity_sum
    <chr>           <int>
  1 CARDINAL_B        768
  2 ORG_B             196
  3 GPE_B             145
  4 PERSON_B          137
  5 DATE_B             48
  6 NORP_B             23
  7 ORDINAL_B           9
  8 LANGUAGE_B          6

  ```
    * Visualisierung 
  ```
  entity_df %>% 
   group_by(entity) %>% 
    summarise(entity_sum = sum(n)) %>% 
    arrange(desc(entity_sum)) %>% 
    filter(entity != "") %>%
    ggplot(., aes(reorder(entity, -entity_sum), entity_sum, fill = entity)) +
    geom_bar(stat = "identity") +
    labs(x = "ENTITY", y = "Anzahl")  +
    ggtitle("Anzahl der Entitäten")
  ```
  
  ![NER_Plot](./Image/NER_Plot.jpeg)
  
  * Extraktion der einzelnen Entitäten ("GPE", "PERSON", "NORP", "ORDINAL", "LANGUAGE","ORG")
  
  ```
  Entitity_df <- corpus_spacyr %>% 
  mutate(ent = map(ner, ~ filter(., str_detect(entity, paste(c("GPE", "PERSON", "NORP", "ORDINAL", "LANGUAGE","ORG"),collapse = '|'))) )) %>% 
  unnest(ent)
  ```

 ### Text Klassifikation
  
  * Berechnng des TF-IDF (Term Frequency – Inversed Document Frequency)
    * Term Frequency: wie häufig ein Wort im gesamten Dokument vorkommt. 
    * Inversed Document Frequency: wie häufig ein Wort nur in bestimmten Dokumenten vorkommt. 
    
  * Berechnung der Cosine- Ähnlichkeit 
    * Anwendung der Funktion pairwise_similarity aus dem Package widyr[3]
   
  * HowTO
  
    * Tokenisierung der Texte 
  ```
  text_tidy <- mycorpus_tidy %>% select(title, text)
    text_tidy %>% 
    slice(1)

  text_tidy %>% 
    unnest_tokens(word, text) %>% 
    anti_join(stop_words)
  ```
  
    * Berechnung der TF-IDF 
  ```
  (text_tfidf <- text_tidy %>% 
    unnest_tokens(word, text) %>% 
    anti_join(stop_words) %>% 
    count(title, word, sort=TRUE) %>% 
    bind_tf_idf(word, title, n))
    
    
    # A tibble: 8,523 x 6
   title                           word         n     tf   idf  tf_idf
   <chr>                           <chr>    <int>  <dbl> <dbl>   <dbl>
 1 british_government_of_burma_act burma      439 0.0307 0.916 0.0282 
 2 british_government_of_burma_act act        409 0.0286 0     0      
 3 british_government_of_burma_act governor   296 0.0207 0.223 0.00462
 4 british_south_africa_act        union      233 0.0293 0.511 0.0150 
 5 constitution_of_burma           union      209 0.0232 0.511 0.0119 
 6 british_south_africa_act        governor   204 0.0256 0.223 0.00572
 7 british_government_of_burma_act person     194 0.0136 0     0      
 8 british_north_american_act      canada     192 0.0314 1.61  0.0505 
 9 british_south_africa_act        council    181 0.0228 0     0      
10 british_government_of_burma_act section    168 0.0118 0     0      
# ... with 8,513 more rows
    
  ```  
  * Berechnung der Cosine- Ähnlichkeit 
  ```
    text_simil<- text_tfidf %>% 
      widyr::pairwise_similarity(title, word, tf_idf) %>% 
      arrange(desc(similarity))
      
      
   # A tibble: 20 x 3
      item1                           item2                           similarity
     <chr>                           <chr>                                <dbl>
    1 constitution_of_burma           british_south_africa_act           0.251  
    2 british_south_africa_act        constitution_of_burma              0.251  
    3 constitution_of_burma           british_government_of_burma_act    0.235  
    4 british_government_of_burma_act constitution_of_burma              0.235  
    5 british_north_american_act      british_south_africa_act           0.134  
    6 british_south_africa_act        british_north_american_act         0.134  
    7 british_south_africa_act        british_government_of_burma_act    0.0584 
    8 british_government_of_burma_act british_south_africa_act           0.0584 
    9 british_north_american_act      constitution_of_burma              0.0397 
   10 constitution_of_burma           british_north_american_act         0.0397 
   11 constitution_of_ireland         constitution_of_burma              0.0376 
   12 constitution_of_burma           constitution_of_ireland            0.0376 
   13 british_north_american_act      british_government_of_burma_act    0.0164 
   14 british_government_of_burma_act british_north_american_act         0.0164 
   15 constitution_of_ireland         british_south_africa_act           0.0102 
   16 british_south_africa_act        constitution_of_ireland            0.0102 
   17 constitution_of_ireland         british_government_of_burma_act    0.00617
   18 british_government_of_burma_act constitution_of_ireland            0.00617
   19 constitution_of_ireland         british_north_american_act         0.00152
   20 british_north_american_act      constitution_of_ireland            0.00152    
  ```

  [^1] : https://en.wikisource.org/wiki/Hong_Kong_Royal_Instructions_1917
  
  [^2] : https://spacy.io/
  
  [3] : https://cran.r-project.org/web/packages/widyr/index.html

