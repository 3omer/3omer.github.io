Part 2: Using Redis to cache most seen articles
To implement this feature first thing we need is to keep track of articles views count. Then we will fetch the top 10 articles and cache them.

Here is the data-structure we gonna use:
1- `sorted-list` of artilces' ids and their views as score
2- `list` to keep the actual top articles after fetching them from DB b
