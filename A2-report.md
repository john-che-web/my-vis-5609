# Assignment 2: Summer Movie Analysis

## Submission Information

- **Demo URL:** `[INSERT DEPLOYED DEMO URL]`
- **GitHub repository:** `[INSERT GITHUB REPOSITORY URL]`

## Dataset

This visualization uses the provided `summer_movies.csv` dataset containing 899 summer movies. The dataset includes release year, runtime, genres, audience rating, vote count, title information, and movie identifiers. Records with missing numeric values remain part of the dataset, while visual channels that require numeric values skip those missing values rather than treating them as zero.

## Reflection

### Why is scatterplot opacity set to 0.5?

The scatterplot uses opacity `0.5` so overlapping points remain visible. When many movies share similar values, partially transparent circles blend together and communicate the density of observations. Areas with many overlapping points appear darker, while isolated observations remain lighter and easier to distinguish.

If every point had opacity `1`, points drawn later would completely cover points drawn earlier. Dense regions would hide observations, making it difficult to tell whether a cluster contains a few movies or many movies. Full opacity would therefore reduce the usefulness of the scatterplot for identifying concentration, overlap, and outliers.

### Which attributes should be removed from the dropdown channels?

The dropdowns should not treat text fields and identifiers as quantitative visual channels. The following attributes should be removed from x-axis, y-axis, and point-size selections:

- `primary_title`
- `original_title`
- `simple_title`
- `tconst`
- `genres`

These fields contain long text, categorical arrays, or identifiers. Mapping them to a numeric position or size does not communicate a meaningful quantitative relationship. `title_type` is also better treated as a categorical filter or grouping variable than as a numeric channel.

Appropriate quantitative channels include `year`, `runtime_minutes`, `average_rating`, and `num_votes`, provided missing values are handled explicitly.

## Guided Exploratory Data Analysis

### Question

**Which factors appear to influence the ratings of summer movies?**

Based on prior intuition, I investigated runtime, number of votes as a proxy for popularity, and genre.

### Runtime

For this analysis, the x-axis is `runtime_minutes`, the y-axis is `average_rating`, and point size is `num_votes`. Point size provides additional context about popularity while the x- and y-positions show the runtime-rating relationship.

![Runtime versus rating](static/images/runtime_vs_rating.png)

*Figure 1. Runtime versus average rating, with point size representing number of votes.*

The relationship between runtime and rating is very weak. The Pearson correlation is approximately `-0.008`, which is effectively zero for this exploratory analysis. The points do not form a strong upward or downward pattern, suggesting that longer movies are not consistently rated higher or lower than shorter movies in this dataset.

### Number of Votes

For this analysis, the x-axis is `num_votes`, the y-axis is `average_rating`, and point size is also based on `num_votes`. The x-position shows popularity directly, while point size emphasizes movies with larger audiences.

![Number of votes versus rating](static/images/num_votes_vs_avg_rating.png)

*Figure 2. Number of votes versus average rating.*

The relationship between vote count and rating is also very weak. The Pearson correlation is approximately `+0.048`. There is a slight positive tendency, but it is not strong enough to conclude that popularity meaningfully predicts rating. The distribution is highly uneven because a small number of movies have many more votes than most movies.

### Genre

Genre shows the clearest differences in average rating. The genre analysis assigns a movie to every genre listed for that movie and compares average ratings across genres.

![Average rating by genre](static/images/rating_by_genre.png)

*Figure 3. Average rating by genre.*

Several genre groups have noticeably different average ratings. Documentary movies have an average rating of approximately `7.31`, while Horror movies average approximately `4.81`. Music movies average approximately `6.65`, and Drama movies average approximately `6.29`.

Genre therefore appears to be the strongest of the three proposed factors. This result should be interpreted as an association rather than proof that genre causes a particular rating. Differences may also reflect audience expectations, sample size, time period, or other characteristics of the movies.

### Guided EDA Conclusion

Runtime and vote count show weak linear relationships with rating in this dataset. Genre shows larger differences between groups, making it the most promising factor for explaining variation in summer movie ratings. The linked visualizations are useful because the scatterplot reveals continuous relationships and outliers, while the genre comparison reveals group-level differences.

## Independent Exploratory Data Analysis

### New Question

**Which genres combine high audience ratings with sustained representation over time?**

This question is different from the guided question because it focuses on the combination of genre quality and genre presence over time rather than testing individual factors against rating alone.

### Visualization and Interaction

The independent analysis contains two linked views:

1. A horizontal bar chart showing the average rating for each genre.
2. A scatterplot showing rating over release year for the selected genre.

Clicking a genre bar updates the time-based scatterplot. The selected-genre summary reports the number of movies and average rating for that genre. Point size uses vote count when it is available, providing additional audience-reach context.

![Independent genre overview](static/images/EDA_charts.png)

*Figure 4. Independent EDA overview showing both linked views.*

![Documentary selected](static/images/EDA_docu.png)

*Figure 5. Documentary selected in the genre analysis.*

![Horror selected](static/images/EDA_horror.png)

*Figure 6. Horror selected in the genre analysis.*

### Independent EDA Analysis

The linked views make it possible to compare group-level rating and temporal distribution. Documentary has a high average rating and can be inspected to determine whether that rating is concentrated in a particular period or appears across many years. Horror has a lower average rating, providing a contrasting view of its rating distribution over time.

The number of movies in each genre is important when interpreting the averages. Genres with very few movies should not be compared with the same confidence as larger groups such as Drama or Comedy. The time view also helps distinguish a consistently represented genre from a genre concentrated in only a few years.

### Independent EDA Conclusion

The independent visualization supports comparison of rating level, sample size, and time coverage in one coordinated view. Genre is not equally represented across the dataset, and the highest-rated genre is not necessarily the most common. Linking the views makes those differences visible and prevents interpreting an average rating without considering when and how often that genre appears.

## Final Summary

The guided analysis found that runtime and vote count have weak relationships with rating, while genre shows larger differences in average rating. The independent analysis extended this finding by linking genre-level ratings to their distribution over time. Together, the views show how coordinated interaction can support both factor comparison and deeper exploratory analysis.