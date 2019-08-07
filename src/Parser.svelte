<script>
  import cheerio from "cheerio";
  import sanitizeHtml from "sanitize-html";
  const data = `<p><a href="http://google.com">google</a> some data <p>Description <strong>some bold text<strong><img src="https://picsum.photos/200/300" /></p><p>some other text<p><a href="#" data-ids="1,2">some link</a><img src="../breeds/hound-basset/n02088238_13799.jpg" /></p>`;
  let html = cheerio.load(data);
  async function parseLinks(html) {
    if (html("a")) {
      await Promise.all(
        html("a")
          .toArray()
          .map(async url => {
            const anchor = cheerio(url);
            if (anchor.data("ids")) {
              const ids = String(anchor.data("ids")).split(",");
              if (ids.length > 1) {
                await Promise.all(
                  ids.map(async id => {
                    const data = await fetch(
                      `https://swapi.co/api/planets/${id}/`
                    );
                    const planet = await data.json();
                    return cheerio(
                      `<a target="_blank" href="${planet.url}">${planet.name}</a>`
                    )
                      .insertBefore(anchor)
                      .css("margin-right", "10px");
                  })
                );
                anchor.remove();
              } else {
                anchor
                  .attr("href", `https://swapi.co/api/planets/${ids[0]}/`)
                  .html();
                anchor.attr("target", "_blank").html();
              }
            } else {
              anchor.attr("target", "_blank").html();
            }
            return url;
          })
      );
    }
    return html.html();
  }
  function parseImgs(html) {
    if (html("img")) {
      html("img").each((index, elem) => {
        const img = cheerio(elem);
        if (!img.attr("src").startsWith("http")) {
          img
            .attr("src", "https://images.dog.ceo" + img.attr("src").slice(2))
            .html();
        }
      });
    }
    return html.html();
  }
  html = parseImgs(html);
  const parsed = parseLinks(cheerio.load(html));
</script>

<style>
  div {
    text-align: justify;
  }
</style>

{#await parsed}
  <p>loading...</p>
{:then markup}
  <div>
    {@html sanitizeHtml(markup, {
      allowedAttributes: Object.assign(
        sanitizeHtml.defaults.allowedAttributes,
        {
          a: ['style', 'href', 'target']
        }
      ),
      allowedTags: sanitizeHtml.defaults.allowedTags.concat(['img'])
    })}
  </div>
{:catch error}
  <p style="color: red">{error.message}</p>
{/await}
