# Nowify 3.0

> A simple app to display your currently playing Spotify track on a Raspberry Pi, made with Vue.

The [original Nowify](https://github.com/jonashcroft/Nowify) was made by user @jonashcroft and his `README.md` is [here](README_original.md) where you can also find a link to the original write up about the project but if you want the updated one, you can look [here](https://ashcroft.dev/blog/nowify-spotify-now-playing-raspberry-pi/).

Nowify allowed a user to authenticate for the Spotify api and displayed the the current track artist, cover, and a matching vibrant background color. It didn't track any data and was fortunately open source. 

I am working on a similar project (More to come later) but wanted a bit more out of it. So now it also displays the context from which you are playing and a [Spotify Code](spotifycodes.com) to let others know where to find it. Be it a playlist, album, artist, or just the song, they can "Scan with Spotify" to find it in the app easily.


Preview:
![Nowify3.0 Preview Image 1](assets/preview-4.png?raw=true "Nowify3.0 preview image, playlist context")
![Nowify3.0 Preview Image 2](assets/preview-5.png?raw=true "Nowify3.0 preview image, album context")
![Nowify3.0 Preview Image 3](assets/preview-6.png?raw=true "Nowify3.0 preview image, artist context")

Nowify3.0 needs a webserver to run. The quickest way to get up and running is to use a Jamstack platform like Netlify or GitHub Pages.

To get it up and running, follow the instructions from @jonashcroft [here](README_original.md)