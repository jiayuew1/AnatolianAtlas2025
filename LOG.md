# AnatolianAtlas
## 5.16 Update
### Site Search and Filtering
- Added a search bar to implement site filtering by AA_ID or site names.
- It will match loosely English Alphabet to Turkish alphabet. E.g. Typing "Hoyuk" still support searching for "Höyük".
- The exact search (from search box), will automatically zoom-in the current selected sites.
- The exact search happen within the selected periods, sites not in selected period(s) won't show up.

### Improved Legend Readability
- Made the legend more readable by combining surveys from the same author but multiple years into one category. The pop-ups and downloaded file will still have the exact year of survey.
- All "Maner_2013", "Maner_2014", etc. now show as just "Maner"
- Kept "Omura" and "Omura Konya" as separate entries

### Selection Behavior
- When selecting a specific site, the map now shows only that site
- Added smooth zooming to selected sites for better navigation
- Implemented a "Clear Selections" button to reset all filters