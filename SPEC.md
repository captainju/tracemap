# TraceMap: specification

## 1. Purpose

TraceMap is a web page that shows an OpenStreetMap map.
The user clicks on the map to trace a route.
The page shows the total length of the route in km.

## 2. Terms

| Term | Meaning |
|---|---|
| Point | A location that the user adds with a click. |
| Segment | The line between two consecutive points. |
| Routed segment | A segment that follows roads and tracks. The routing service calculates it. |
| Straight segment | A segment that is a straight line between two points. |
| Route | The ordered list of points and the segments between them. |
| Closing segment | A routed segment from the last point back to the first point. |
| Closed loop | A route that has a closing segment. |
| Favorite | A route that the user saves with a name, in the favorites list. |
| Current route | The route on the screen. |
| Unsaved route | A current route that has at least one point, and that the user never saved, or that changed after the last save or load. |

## 3. Technical constraints

- The page is one file: `index.html`. It contains the HTML, CSS, and JavaScript.
- The page uses Leaflet, loaded from a CDN. Do not add other libraries.
- The page has no build step and no back end.
- The page has a favicon: a white route line with a green start dot, on the blue route color. It is an inline SVG (`data:` URL), so the page stays one file.
- The user opens `index.html` directly in the browser (a `file://` URL). A static web server also works, for example `python3 -m http.server`.
- The `localStorage` data (route, favorites, choices) belongs to one page address. The `file://` page and a page on a web server do not share it. Use the favorites Export and Import to move the favorites.
- The page works in current versions of Firefox and Chrome on a desktop computer.

## 4. Map

- The map has three backgrounds, all with OpenStreetMap data. A layer button at the bottom right selects one:

| Background | Tile URL | Use |
|---|---|---|
| OSM France (default) | `https://{s}.tile.openstreetmap.fr/osmfr/{z}/{x}/{y}.png` | General map, close to the standard OSM map. Its tiles stop at zoom 18. At zoom 19, the page enlarges the zoom 18 tiles. |
| CyclOSM | `https://{s}.tile-cyclosm.openstreetmap.fr/cyclosm/{z}/{x}/{y}.png` | Shows paths, tracks, and their surface. |
| OpenTopoMap | `https://{s}.tile.opentopomap.org/{z}/{x}/{y}.png` | Contour lines and hill shading. Its tiles stop at zoom 17. Above, the page enlarges the zoom 17 tiles. |

- The page keeps the selected background in `localStorage`.
- Each background shows the OpenStreetMap attribution and the attribution of the tile service.
- Do not use `https://tile.openstreetmap.org`. This server blocks a page opened as a file, because the browser sends no `Referer` header. The answer has the header `x-blocked: No referer sent`.
- At first load, if a home is set (see section 4.1), the map center is the home and the zoom level is 16.
- At first load, if no home is set, the map shows all of metropolitan France (latitude 41.3 to 51.1, longitude -5.2 to 9.6). The fit has no animation.
- If a saved route exists (see section 9), the map fits the view to that route instead, with a margin of 30 pixels and a zoom of 18 or less. The fit has no animation.
- The user can pan and zoom the map with the usual Leaflet controls.
- On the map, the mouse cursor is a crosshair. Do not use the Leaflet "grab" cursor: some Linux cursor themes do not have it, and Chrome then shows no cursor.
- The zoom control (+ and −) is at the bottom right of the map. The favorites panel uses the left side (see section 10).

### 4.1 Home icon

- By default, no home is set. The map shows no home icon.
- The **Set home** button starts the home selection. The button text changes to **Cancel**, and the status shows "Click on the map to set the home".
- During the home selection, the next left click on the map sets the home at that location. It does not add a point.
- A click on **Cancel** stops the home selection.
- The **Remove home** button removes the home and its icon. When no home is set, the button is disabled.
- The page keeps the home in `localStorage` (key `tracemap.home`, value `[lat, lng]`).
- When a home is set, the map shows a "home" icon at that location.
- The icon is the 🏠 emoji in a Leaflet `divIcon`. Do not add an image file.
- The home icon is not a point. It is not part of the route, the length, or the GPX file.
- The user cannot drag the home icon.
- The home icon does not catch clicks. A click on it adds a point at that location, as on the rest of the map.
- Undo and reset do not remove the home icon.

## 5. Tracing the route

### 5.1 First point

- A click with either mouse button on the map adds the first point.
- The first point has no segment.

### 5.2 Next points

- A **right click** adds a point and a **routed segment** from the previous point.
  - The page puts the new point on the nearest path, at the end of the BRouter route. The clicked point is not kept.
  - The first point of a route stays at the clicked location: there is no route yet.
- A **left click** adds a point and a **straight segment** from the previous point.
- The page blocks the browser context menu on the map. A right click must not open it.
- A click with either mouse button on a point or on a handle does not add a point.

### 5.3 Routed segments

- The routing service is BRouter: `https://brouter.de/brouter`.
- The routing profile comes from the profile selection (see section 5.6).
- Put the service URL and the profiles in constants at the top of the script.
- The request format is GeoJSON: `?lonlats=LON1,LAT1|LON2,LAT2&profile=PROFILE&alternativeidx=0&format=geojson`.
- BRouter starts and ends its route on the nearest path. The page joins these ends to the clicked points:
  - If a clicked point is 5 m or less from the end of the BRouter route, the clicked point replaces that end. The line has no detour.
  - Otherwise, the page adds a straight line from the end of the BRouter route to the clicked point.
  - Put the 5 m distance in a constant.
- While the request runs, the page shows a "routing..." status.
- While the request runs, the page ignores new clicks on the map.
- If the request fails, the page does not add the point. It shows an error message for 5 seconds.

### 5.4 Display

- Each point shows as a marker on the map: a white disc with a blue border. The first point has a green border `#2e7d32`.
- A routed segment shows as a solid line.
- A straight segment shows as a dashed line.
- Both segment types use the same color.
- The closing segment shows as a solid line, like the other routed segments.
- An arrow every 500 m along the route shows the direction of the route. The first arrow is at 500 m from the first point.
- Each arrow follows the direction of the route at its location. The arrows are white, with a border in the color of the route, so that they show on the line.
- The arrows do not catch clicks. Put the distance in a constant.
- The control panel has a "Show direction arrows" check box. It shows or hides the arrows. It is on by default.
- The page keeps the state of the check box in `localStorage`. After a reload, the arrows keep the same state.
- The arrows update with the lines, also during a drag.
- A km marker shows at each full kilometer along the route: 1, 2, 3, and so on.
- A km marker is a white disc with a border in the color of the route, and the number in bold.
- The km markers do not catch clicks.
- A km marker must not hide a route point. If a km marker is less than 20 pixels from a route point on the screen, it moves 18 pixels away from its location. Put these values in constants.
- The page tries 8 directions (each 45°) and uses the direction that is farthest from the route lines. At a corner, the marker then goes outside the corner, not on a line.
- The page checks this again after each zoom. During a drag or a routing request, the check waits for the next redraw.
- The control panel has a "Show km markers" check box. It is on by default. The page keeps its state in `localStorage`.
- When the km markers show, the page does not draw an arrow at the location of a km marker.
- The km markers update with the lines, also during a drag.

### 5.5 Closing the loop

- The Close loop button adds a closing segment. It does not add a new point.
- The closing segment is always a routed segment. It uses the same request as in section 5.3.
- If the request fails, the route stays open. The page shows an error message for 5 seconds.
- When the loop is closed, the page ignores clicks on the map. No point can be added.
- To add more points, the user clicks Undo to open the loop again.
- The Close loop button is disabled when the route has fewer than 2 points or the loop is closed.

### 5.6 Routing profile

- The control panel has a "Routing" selection with two profiles:

| Profile | BRouter profile | Use |
|---|---|---|
| Trail (default) | `hiking-mountain` | Follows trails, tracks, and small roads. |
| Road | `trekking` | Prefers paved roads and cycle lanes. |

- The selected profile applies to each new routed segment.
- Each routed segment keeps the profile that calculated it. When the page recalculates a segment, it uses the same profile.
- A routed segment saved before this feature uses the Trail profile.
- The page keeps the selected profile in `localStorage`.

## 6. Moving a point

- The user can drag any point to a new location.
- A drag of 5 pixels or less does nothing: the point goes back to its location. A hand moves a little during a double-click, and the double-click must still work.
- During the drag, the segments that touch the point follow the pointer at once. The total length updates too.
- During the drag, the page also recalculates the routed segments that touch the point. To limit the load on BRouter:
  - The page sends a request only when the point stays within 5 pixels for 300 ms.
  - The page does not send a request if the point is within 5 pixels of the last requested location.
  - Only one recalculation runs at a time. A recalculation sends one request for each routed segment that touches the point, at the same time.
  - Put the delay and the distance in constants at the top of the script.
- BRouter limits the number of requests. When the limit is reached, it answers "Please, retry later!".
- When the user drops the point, the page recalculates the segments that touch this point. If the point is within 5 pixels of the last requested location, the page uses the last result.
- Each recalculated segment keeps its type (routed or straight).
- In a closed loop, the closing segment touches the first point and the last point. It updates when the user moves one of them.
- If a routing request fails, the point goes back to its old location. The page shows an error message.

### 6.1 Inserting a point

- Each segment shows a handle: a small faded dot at the middle of the segment, along its geometry.
- The closing segment also has a handle.
- The handles show below the points. A handle never hides a point.
- The user drags a handle to insert a new point at the drop location.
- The new point splits the segment into two segments. The mouse button of the drag gives their type, as in section 5.2:
  - A drag with the left button makes two straight segments.
  - A drag with the right button makes two routed segments. If the old segment was routed, they keep its profile. Otherwise, they use the selected profile.
- A drag with the right button does not open the browser context menu, and it does not add a point at the drop location.
- During the drag, the two segments behave as in section 6: live lines, and routing requests after a pause.
- If a routing request fails, the page does not insert the point. It shows an error message.

### 6.2 Removing a point

- The user double-clicks a point with the left mouse button to remove it.
- For a point in the middle of the route, a new segment joins the two neighbours of the removed point.
- The new segment takes the type of the segment that arrived at the removed point.
- For the first point, the page removes the first segment. The second point becomes the start.
- For the last point, the page removes the last segment.
- In a closed loop, if the user removes the first or the last point, the page recalculates the closing segment.
- If fewer than 2 points remain, the closing segment disappears.
- If a routing request fails, the page does not remove the point. It shows an error message.

## 7. Total length

- The total length shows at the top middle of the screen, outside the control panel.
- The total length does not catch clicks. A click on it adds a point, as on the rest of the map.
- The page always shows the total length of the route in km, with 2 decimals. Example: `12.34 km`.
- The page calculates the length from the segment geometry with the haversine formula.
- The page uses the same calculation for routed and straight segments.
- With zero or one point, the total length is `0.00 km`.
- The total length updates after each change: add, move, close loop, undo, reset, load.
- The box at the top middle has two lines. The first line shows the length, the climb, and the descent (section 7.1).
- The second line, in a smaller font, shows the distance to the start, the time estimate, and the surface split (sections 7.2 to 7.4). It shows only the items that apply.
- Example of a second line: `↩ 2.30 km to start · ≈ 1 h 05 min · 62 % trail · 38 % road`.

### 7.1 Climb and descent

- The page shows the total climb (D+) and the total descent (D-) next to the total length, in whole meters. Example: `12.34 km · D+ 120 m · D- 80 m`.
- The elevation comes from BRouter. BRouter gives an elevation for each coordinate of a routed segment.
- BRouter gives the first and the last coordinate of its route the elevation of a node of the way, up to several hundred meters away. The page removes the elevation of these two coordinates. It then interpolates their elevation between the nodes on each side.
- Without this rule, each route point shows a false climb or descent of a few meters in the profile.
- The page does not use another elevation service.
- A coordinate without elevation gets a linear interpolation along the distance, between the nearest coordinates with elevation.
- Before the first coordinate with elevation, and after the last one, the page copies the nearest elevation.
- The page ignores elevation changes smaller than 5 m, to remove the noise of the elevation data. Put this value in a constant.
- With zero or one point, the page shows `D+ 0 m · D- 0 m`.
- If the route has segments but no coordinate with elevation, the page shows `D+ – · D- –`. Example: a route with only straight segments, or a route saved before this feature.
- The climb and the descent update at the same time as the total length, also during a drag.

### 7.2 Distance to start

- The page shows the straight-line distance from the last point to the first point, in km with 2 decimals. Example: `↩ 2.30 km to start`.
- The page shows it only when the route has at least 2 points and is not a closed loop.

### 7.3 Time estimate

- The control panel has a "Pace" field, in minutes and seconds per km. Example: `5:30`. The default value is `6:00`.
- The page calculates the effort distance: the length in km, plus 1 km for each 100 m of climb. This is the ITRA "km-effort" rule.
- The time estimate is the pace multiplied by the effort distance. If the route has no elevation, the page uses the length only.
- The page shows the estimate rounded to the minute. Examples: `≈ 45 min`, `≈ 1 h 05 min`.
- If the pace is not valid, the field shows a red border, and the page shows no estimate.
- The page keeps the pace in `localStorage`.

### 7.4 Surface split

- For each routed segment, BRouter gives the way tags of each part of the segment, with its length.
- The page sorts each part into one of three classes:

| Class | Rule |
|---|---|
| Trail | `surface` is unpaved, gravel, fine_gravel, compacted, dirt, earth, ground, grass, mud, sand, woodchips, pebblestone, or rock. Without `surface`: `highway` is track, path, or bridleway. |
| Road | `surface` is paved, asphalt, concrete, paving_stones, sett, or cobblestone. Without `surface`: any other `highway` value. |
| Unknown | Straight segments, routed segments saved before this feature, segments from a GPX import, a `surface` value that is in neither list, and a part without `surface` and without `highway`. |

- The page shows the share of each class in the total length, in whole percents. Example: `62 % trail · 38 % road`.
- The page shows the Unknown class only if it is 1 % or more.
- If no part of the route is Trail or Road, the page shows no surface split. Example: a route with only straight segments.
- Each routed segment stores its lengths per class. The favorites keep them.

### 7.5 Elevation profile

- The control panel has a "Show elevation profile" check box. It is on by default. The page keeps its state in `localStorage`.
- The profile is a strip at the bottom of the map, 150 pixels high. It goes from the left edge to 10 pixels before the map buttons at the bottom right (layer button and zoom control).
- The strip shows only when the check box is on and the route has at least one coordinate with elevation.
- The page draws the chart in SVG, without a chart library.

Chart:

- The horizontal axis is the distance from the first point, in km. The vertical axis is the elevation, in meters.
- The page samples the route at regular distances: one sample for each pixel of the chart width. It interpolates the elevation of each sample along the distance.
- The elevation is a 2-pixel line in dark blue `#0d366b`. The parts without real elevation (interpolated, see section 7.1) show as a dashed gray line `#898781`.
- The area below the line shows the slope of each sample (see the slope colors below). The parts without real elevation also have this area, with the slope of the interpolated elevation.
- The vertical axis shows the minimum and the maximum elevation, rounded to 10 m, with hairline gridlines `#e1e0d9`. The axis covers at least 20 m, so that a flat route does not look steep.
- The horizontal axis shows a distance label (no tick mark) each 1 km up to 15 km, each 2 km up to 30 km, and each 5 km above.
- The axis text is gray `#898781`, 11 pixels. Text never uses the data colors.

Slope colors:

- The color of the area below the line shows the steepness of each sample. The steepness is the absolute slope over 50 m (25 m before and 25 m after the sample), calculated from the full route geometry, at any sample spacing.
- The colors use one blue hue from light to dark. Red and green are not used, because some color-blind users cannot see the difference.
- The color changes gradually with the slope. The page fills the area with a horizontal gradient, with at most 600 color stops at regular distances.
- Each slope class has one color. Near the limit between two classes, the color blends over 1 % on each side of the limit. Put this value in a constant.

| Slope | Color |
|---|---|
| Less than 5 % | `#86b6ef` |
| 5 % to 10 % | `#2a78d6` |
| More than 10 % | `#104281` |

- Example: from 4 % to 6 %, the color changes gradually from `#86b6ef` to `#2a78d6`. From 6 % to 9 %, it stays `#2a78d6`.

- A legend at the top right of the strip shows the gradient from 0 % to 15 %.

Hover and click:

- When the pointer is on the chart, a vertical hairline follows the nearest sample. A red dot `#d03b3b` (12 pixels, white ring) shows on the line.
- A tooltip shows the elevation first, in bold, then the distance and the slope. Example: **245 m** · 3.20 km · +8 %. For an interpolated part, the slope starts with ≈.
- At the same time, a red dot `#d03b3b` (14 pixels, white ring) shows on the route on the map, at the same location. This dot is above all other items on the map: points, handles, arrows, and km markers.
- When the pointer is within 12 pixels of the route line on the map, the chart shows the hairline and the tooltip for the nearest location on the line. The red dot shows at that location.
- When the pointer is on a route point or on a handle, the chart still shows the hairline and the tooltip, but the map shows no red dot. The red dot would hide the point or the handle.
- This does not change the clicks on the map. A click near the route still adds a point.
- A click on the chart moves the map to center on that location. The zoom does not change.
- The strip does not pass clicks, scroll, or drags to the map.

Updates:

- The chart updates with the lines, also during a drag. It redraws at most once per screen frame.
- The chart redraws when the browser window changes size.

## 8. Controls

The page shows a control panel at the top right of the map. The panel contains these items:

| Control | Action |
|---|---|
| Undo button | Cancels the last action. |
| Reset button | Removes the full route. |
| Close loop button | Adds a closing segment (see section 5.5). |
| Out and back button | Adds the same route back to the start (see section 8.5). |
| Import GPX button | Replaces the route with the track of a GPX file (see section 8.6). |
| Export GPX button | Downloads the route as a GPX file. |
| Set home and Remove home buttons | Set or remove the home (see section 4.1). |
| Routing selection | Selects the routing profile (see section 5.6). |
| Pace field | Sets the pace for the time estimate (see section 7.3). |
| Check boxes | Show direction arrows, Show km markers, Show elevation profile. |
| Search box | Moves the map to a place name. |
| Status line | Shows "routing...", the messages, and the errors. |
| Help text | Explains the mouse actions: left click, right click, drag, double-click. |

While a routing request runs, these controls are disabled: Undo, Reset, Close loop, Out and back, Save, and Save as new. Export GPX and Import GPX stay enabled. A GPX file selected during a request is ignored.

Messages:

- While the favorites panel is open, the messages show in the panel, not in the control panel.
- An error message shows for 5 seconds. Examples: "Routing failed: ...", "Search failed: ...", "Import failed: ...".
- After a GPX import, the page shows "Imported 21.61 km with 23 points" for 5 seconds.

The eight buttons are in a grid of two columns, in this order:

| Left | Right |
|---|---|
| Undo | Reset |
| Close loop | Out and back |
| Import GPX | Export GPX |
| Set home | Remove home |

### 8.1 Undo

- Undo cancels the last action. An action is "add a point", "move a point", "insert a point", "remove a point", "close the loop", or "out and back".
- For "add a point", undo removes the last point and its segment.
- For "move a point", undo puts the point back to its old location and restores the old segments.
- For "close the loop", undo removes the closing segment. The route is open again.
- For "insert a point" and "remove a point", undo restores the route as it was before the action.
- Undo does not send a new routing request. It restores the stored geometry.
- The undo history keeps all actions since the last page load, reset, favorite load, or GPX import. Each of these clears the history.
- The Undo button is disabled when the history is empty.
- The Back button of the browser runs Undo. The Back button of the mouse and the Alt+← keys do the same, because they also go back in the browser history.
- To do this, the page adds one entry to the browser history while the undo history is not empty. Back removes this entry. The page then runs Undo and adds the entry again.
- When the undo history is empty, Back goes to the previous page, as usual.
- While a routing request runs, Back does nothing.

### 8.2 Reset

- Reset removes all points and segments, and clears the undo history.
- The first click on Reset changes the button text to "Confirm reset?".
- A second click within 3 seconds removes the route. After 3 seconds, the button goes back to "Reset".
- Do not use the browser `confirm()` dialog. Some browsers block it, and it then returns false.
- Reset also deletes the saved route (see section 9).
- The Reset button is disabled when the route is empty.

### 8.3 GPX export

- The button downloads a GPX 1.1 file with one track (`<trk>`) and one track segment (`<trkseg>`).
- Each track point has an `<ele>` element with the elevation from section 7.1. If the route has no elevation, the track points have no `<ele>` element.
- The track contains all coordinates of all segments, in order. It does not repeat shared points.
- For a closed loop, the track includes the closing segment. The track ends at the first point.
- The file name is `tracemap-YYYYMMDD-HHMM.gpx`, with the local date and time.
- If the current route has a favorite (see section 10), the file name starts with the favorite name instead of `tracemap`. Example: `Boucle du Grand Large-20261018-1604.gpx`. This is also true when the route has changes after the load.
- In the favorite name, the page replaces the characters that a file name cannot contain (`\ / : * ? " < > |` and the control characters) with `_`. It also removes the spaces at the start and at the end.
- The track name (`<trk><name>`) is the favorite name, or `TraceMap` if the route has no favorite.
- The Export GPX button is disabled when the route has fewer than 2 points.

### 8.4 Address search

- The search uses the Nominatim service: `https://nominatim.openstreetmap.org/search`.
- The user types a place name and presses Enter or clicks a search button.
- Do not send a request on each key press. The Nominatim usage policy forbids it.
- The page moves the map to the first result. It does not add a point.
- If there is no result, the page shows "No result" for 5 seconds.

### 8.5 Out and back

- The Out and back button adds the route in the opposite direction, from the last point back to the first point.
- The new points are copies of the existing points, in the reverse order. The last new point is at the location of the first point.
- Each new segment is a copy of an existing segment, in the reverse direction, with the same type, geometry, and surface data.
- The page sends no routing request for this action.
- The route is not a closed loop after this action. The user can add more points.
- Undo removes all the new points in one step.
- The Out and back button is disabled when the route has fewer than 2 points, or when the loop is closed.

### 8.6 GPX import

- The Import GPX button opens a file selection for a `.gpx` file.
- If the current route is an unsaved route (see section 10.4), the first click changes the button text to "Replace the unsaved route?". A second click within 3 seconds opens the file selection.
- The page reads the track points (`<trkpt>`) of all tracks, in order. If the file has no track point, it reads the route points (`<rtept>`).
- The page reads the elevation (`<ele>`) of each point, if it exists. The climb and the descent then use these elevations.
- A GPX file from an older TraceMap version has a detour at each route point: point A on the path, the clicked point, then point A again. If two track points at the same location surround a third point that is 25 m or less away, the page removes the third point and the second copy of A. A longer detour is a real out and back, and the page keeps it. It also removes the elevation of A, and interpolates it.
- The page keeps the full geometry of the track. It removes a track point at the same location as the point before it.
- The page adds a route point at the first track point after each full km, and at the last track point. Put the distance in a constant.
- It adds no route point less than 250 m (a quarter of the distance) before the last track point, so that the last segment is not too short.
- Each part of the track between two route points becomes a routed segment with the Trail profile, without a routing request.
- If the user moves a route point, the page recalculates the two segments that touch it with BRouter. The rest of the track keeps its geometry.
- The surface split shows these segments as Unknown.
- The imported route replaces the current route. It clears the undo history, and the map fits the view to the route.
- The imported route has no favorite. It is an unsaved route.
- The name of the track (`<trk><name>` or `<metadata><name>`), if it exists, is the default name for "Save as new". The page keeps this name in memory only: after a page load, a reset, a favorite load, or a save, the default is the date again.
- If the file is not valid GPX, or has fewer than 2 points, the page changes nothing. It shows an error message for 5 seconds.
- GPX waypoints (`<wpt>`) are ignored.

## 9. Save in the browser

- The page saves the route in `localStorage` after each change.
- If `localStorage` is full, the route stays on the screen but is not saved. The page shows "Storage full: the route is not saved. Delete or export favorites" for 5 seconds.
- The saved data contains the points, the segment types, the segment geometry with the elevations, and the closing segment, if it exists.
- At page load, the page restores the saved route without new routing requests.
- The page does not save the undo history. After a reload, the history is empty.
- The saved data also contains the link to the favorite of the current route, if it exists, and the unsaved state.
- If the saved data is not valid, the page ignores it and starts with an empty route.
- Routes saved by an older version have, at each junction of two routed segments, the same path location at the end of the first segment and at the start of the second, with two different elevations. When the page loads a route or the favorites, it removes these two elevations, and interpolates them.

## 10. Favorites

### 10.1 Storage

- The page keeps the favorites in `localStorage`, in a key that is different from the current route key.
- Each favorite contains an identifier, a name, the date of the last save, and the route data from section 9.
- The page does not store the length, the climb, or the thumbnail. It calculates them from the route data.
- `localStorage` holds about 5 MB. This is about 80 to 100 routes of 20 km.
- If `localStorage` is full, the page does not save. It shows "Storage full: delete or export favorites" for 5 seconds.
- Favorites belong to one browser and one page address. Another browser or another port does not see them.

### 10.2 Favorites panel

- A "★ Favorites" button is at the top left of the map. The button opens the favorites panel.
- While the favorites panel is open, the button is hidden. The close button (✕) of the panel shows it again.
- The favorites panel is on the left side of the screen, over the map. It has a close button (✕).
- The page keeps the state of the panel (open or closed) in `localStorage`. After a page load, the panel has the same state.
- The map stays usable while the panel is open. Clicks and scroll on the panel do not reach the map.
- From top to bottom, the panel contains these items:
  1. The save area (see section 10.3).
  2. A search box.
  3. The list of favorites.
  4. The Export and Import buttons (see section 10.8).
- The list shows the favorites sorted by date of last save, newest first.
- The search box filters the list by name, while the user types. The filter ignores case and accents. Example: "grand" finds "Boucle du Grand Large".
- If there is no favorite, the panel shows "No favorite yet". If the search finds no favorite, the panel shows "No favorite matches the search".

### 10.3 Saving

- The save area shows the name of the favorite of the current route, if it exists. Otherwise, it shows "Route not saved" (the route has points) or "No route".
- If the current route is an unsaved route, the save area shows "(modified)" after the name.
- The "Save" button updates the favorite of the current route: its route data and its date. The button shows only when the current route has a favorite.
- The "Save as new" button creates a new favorite with the current route.
- After "Save as new", a name field shows in the save area. Its default value is the name of the last imported GPX track, if the current route comes from a GPX import (see section 8.6). Otherwise, it is "Route DD/MM/YYYY HH:MM", with the local date and time.
- The user presses Enter or clicks OK to save. The user presses Escape or clicks Cancel to cancel.
- The page refuses an empty name, and shows "The name cannot be empty". A name has 100 characters or less. Two favorites can have the same name.
- The page refuses to save a route with fewer than 2 points, also when the name field is still open. It shows "A favorite needs at least 2 points".
- After a save, the current route is linked to the saved favorite, and it is not an unsaved route.
- Both buttons are disabled when the current route has fewer than 2 points.
- Do not use the browser `prompt()` dialog. Some browsers block it.

### 10.4 Unsaved state

- Each action from sections 5, 6, and 8 that changes the route makes it an unsaved route. Undo also does.
- Reset removes the link to the favorite. The current route is then empty and not an unsaved route.
- A route from before this feature has no favorite. If it has points, it is an unsaved route.

### 10.5 List item

Each item of the list shows:

- A thumbnail (see section 10.6).
- The name.
- The length and the climb, as in section 7. Example: `12.34 km · D+ 320 m`.
- The date of the last save, in the format DD/MM/YYYY.
- The ↻ sign, if the route is a closed loop.
- A rename button (✎) and a delete button (🗑).

The item of the favorite of the current route has a highlight.

### 10.6 Thumbnail

- The thumbnail is an SVG drawing of the route line, 96 × 64 pixels, on a plain background. It does not show the map.
- The drawing keeps the shape of the route: the horizontal scale uses the cosine of the latitude.
- The drawing fits the route in the thumbnail, with a margin of 4 pixels.
- A green dot shows the first point.
- The page draws the thumbnail when it shows the list. It does not store the drawing.

### 10.7 Loading, renaming, and deleting

- A click on the thumbnail or on the name of an item loads the favorite.
- If the current route is an unsaved route, the item first shows a message: "Replace the unsaved route?", with the buttons "Replace" and "Cancel".
- Loading replaces the current route, clears the undo history, and fits the map view to the route.
- After a load, the current route is linked to the loaded favorite, and it is not an unsaved route.
- The rename button changes the name into a field. Enter or a click outside the field saves the new name. Escape cancels. An empty name keeps the old name, and the page shows "The name cannot be empty".
- A rename does not change the date of the last save.
- The first click on the delete button changes it into "Delete?". A second click within 3 seconds deletes the favorite.
- If the user deletes the favorite of the current route, the current route stays on the screen. It loses its link, and it becomes an unsaved route.

### 10.8 Export and import

- The Export button downloads all favorites in one JSON file: `tracemap-favorites-YYYYMMDD.json`.
- The Import button opens a file selection. The user selects a file that the Export button made.
- The import adds each favorite of the file to the list. It does not delete any favorite.
- If a favorite of the file has the same identifier as a favorite of the list, the page keeps the favorite with the newest date of last save.
- After the import, the page shows the result for 5 seconds. Example: "3 added, 1 updated, 2 ignored".
- If the file is not valid, the page changes nothing. It shows an error message for 5 seconds.

## 11. Out of scope

These items are not part of this version:

- Touch devices. A touch screen has no right click.
- Dark mode for the elevation profile. The page has no dark mode.
- Elevation for straight segments from an elevation service.
- More than one route at a time.
- Synchronization of favorites between browsers or computers. Use Export and Import.
- Folders or tags for favorites.
- Points of interest on the map, for example drinking water or toilets.

## 12. Acceptance checks

1. Double-click `index.html` with an empty `localStorage`. The map shows all of France, with no home icon.
2. Left click two times. A dashed line connects the two markers. The length is more than 0.
3. Right click one time near a road. A solid line follows the road to the new marker.
4. Right click on the map. The browser context menu does not open.
5. Drag the middle marker. During the drag, the two segments that touch it and the length update. After the drop, they show the final route.
6. Click Undo. The marker goes back. Click Undo again. The last point disappears.
7. Reload the page. The route and the length are the same as before the reload.
8. Click Export GPX. A `.gpx` file downloads. Another tool, for example gpx.studio, opens it.
9. Click Close loop. A solid line follows the paths from the last marker to the first marker. The length increases. A click on the map adds no point. Click Undo. The loop opens again.
10. Drag the handle of a routed segment with the right button. A new point appears where you drop it, with two routed segments. Click Undo. The point disappears. Drag the same handle with the left button. The two new segments are straight lines.
11. Double-click a middle point. A new segment joins its two neighbours. Click Undo. The point comes back.
12. Search "Lyon". The map moves to Lyon. No point is added.
13. Click Reset, then click Confirm reset?. The map shows no route. The length is `0.00 km`. A reload shows no route.
14. Trace a route. Open Favorites. Click Save as new, type "Test", press Enter. The list shows "Test" with a thumbnail, the length, D+, and today's date.
15. Move a point. The save area shows "Test (modified)". Click Save. "(modified)" disappears.
16. Click Reset and confirm. Trace another route. Click the "Test" item. The page asks "Replace the unsaved route?". Click Replace. The "Test" route shows.
17. Rename "Test" to "Boucle du Grand Large". Type "grand" in the search box. The item stays in the list.
18. Click Export. A JSON file downloads. Delete the favorite. Click Import and select the file. The favorite comes back.
19. Trace a 3 km route with right clicks (routed segments). The markers 1, 2, and 3 show. The second line shows the distance to the start, the time, and the surface split.
20. Set the pace to 5:00. The time estimate changes. Type "abc". The field gets a red border and the estimate disappears.
21. Click Out and back. The length doubles. The distance to the start is 0.00 km. Click Undo. The route is as before.
22. Select Road. Right click on the map. The new segment prefers paved roads. Drag a point of an older Trail segment. That segment still uses Trail.
23. Import a GPX file of a 10 km run while the route on the screen is unsaved. The button asks "Replace the unsaved route?". Click again and select the file. The track shows with 11 points (the start, one each full km, and the end), and the length and D+ of the file. Click Save as new: the name field shows the track name.
24. Trace a route with a climb. The elevation profile shows, with the area colored by slope (gradual colors) and its legend. Move the pointer on the chart: the tooltip and a dot on the map follow. Move the pointer along the route on the map: the chart hairline follows. Click on the chart: the map centers on that location.
25. Click Set home, then left click on the map. The home icon shows there, and no point is added. Reload the page with no route. The map centers on the home at zoom 16. Click Remove home. The icon disappears.
