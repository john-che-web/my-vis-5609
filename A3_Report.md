# A3 Report: Summer Movies 3D Visualization

## Submission Links

- **Demo:** [Add the deployed A3 demo URL here](PASTE_DEMO_URL_HERE)
- **GitHub repository:** [Add the GitHub repository URL here](PASTE_GITHUB_REPOSITORY_URL_HERE)

## Screenshots

### A1 2D Bar Chart

![A1 2D bar chart](misc/A1%20Bar%20chart.png)

### A3 3D Visualization

![A3 3D visualization](misc/A3_5.png)

The A3 scene includes animated flamingos and a parrot flying across the scene, as well as horses running in place while orbiting around the visualization. The Parrot model was modified by changing its scale, position, speed, and animation duration.

## Reflection: 3D Compared with 2D

The A1 visualization uses a conventional 2D stacked bar chart to show how movie genres change over time. The A3 visualization represents each movie count as an individual 3D object and places the objects in a 3D environment with lighting, textures, shadows, and animation.

### Pros

The 3D visualization is more expressive and engaging than the A1 chart. The animated birds and horses create a stronger sense of movement and make the visualization feel interactive rather than static. The scene also provides more opportunities for visual storytelling through the sky, ground, models, lighting, and depth.

The individual 3D units make the counts tangible. Instead of seeing only the height of a colored bar, the viewer can see separate objects accumulating into a stack. The different geometries and colors also provide a direct visual distinction between Comedy, Romance, and Drama. The 3D scene can therefore be more memorable and engaging for an audience experiencing the visualization as an interactive demonstration.

### Cons

The A1 2D chart is more legible and efficient for exact comparison. Its aligned baseline, regular axes, and consistent bar widths make it easier to compare values across years. The A3 visualization introduces perspective, depth, lighting, shadows, and occlusion. Objects in the foreground can hide objects behind them, and the apparent size of objects changes with depth. These effects make it harder to estimate counts precisely.

The A3 visualization is also more cluttered than the A1 chart. The 2D chart communicates the trend with a relatively small number of colored bar segments, while A3 displays many individual objects across dozens of years. The large number of units can make the stacks visually dense, especially in years with many movies. The 3D models and animated objects add additional visual elements that are not part of the data.

Animation can also be a distraction. The flying birds and orbiting horses draw attention away from the movie stacks, particularly when the viewer is trying to compare values or read labels. Their movement is useful for making the scene feel alive, but it does not improve the core year-by-genre comparison. The animation may therefore increase engagement while reducing focus and making a precise reading more difficult.

The genre-specific shapes are also not strictly necessary for communicating the differences between genres. Color and the legend already distinguish Comedy, Romance, and Drama, and the 2D chart demonstrates that color can communicate the categories effectively. Using spheres, cones, and cylinders adds visual interest, but it also increases the amount of information the viewer must decode and can make the stacks more difficult to compare.

### Proposed Solution

A practical solution is to preserve the expressive 3D scene while making the data labels and supporting layout more legible. The implementation applies several changes to address the visibility problems:

- Invalid year values such as `NA` are excluded from the aggregation.
- Year labels are spaced out instead of displaying every year, using five-year intervals plus the first and last years.
- Year labels are enlarged, rendered in white, and given a dark contrast backing.
- Year labels are positioned above their corresponding stacks so the relationship between a date and its data is clearer.
- Genre units are enlarged moderately without allowing them to become excessively crowded.
- Genre shapes and legend text are enlarged so the encoding is easier to decode.
- The legend is moved to the upper-right, outside the tallest stacks, and uses black text against the light sky for contrast.

A further practical improvement would be to provide an animation toggle or pause-on-hover behavior. This would allow viewers to enjoy the expressive movement when desired but pause the scene when they need to compare values or read labels. Another possible improvement would be to offer a 2D mode or synchronized 2D overview for precise analysis alongside the 3D presentation.

## Data Aggregation

The visualization loads `summer_movies.csv`, filters invalid years, and groups the remaining movies by year. For each year, it counts movies whose comma-separated genre list contains Comedy, Romance, or Drama. Each count becomes a vertical stack of individual 3D units. A movie with multiple target genres contributes once to each matching genre count.

## GenAI Attribution

Codex was used as a GenAI coding assistant during this assignment. It assisted with repository inspection, Three.js/Svelte implementation, CSV aggregation, unit visualization, animation configuration, label and legend layout, debugging, and report drafting. The project decisions, visual review, and final direction were provided by the student.