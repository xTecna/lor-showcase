FYI [2]: Due to this project being comissioned by Riot Games, it will be no longer available to the public as that was meant to be a private project.

FYI: This project was comissioned by Riot Games to present the spoiler season cards in a timely fashion. We use this <a href="https://www.notion.so/LoR-Spoiler-Cards-Showcase-d3ef797658474378b5eda57ea6b35b22">Notion document</a> to present them all the information pertaining this project. Below is a direct copy for the contents shown in this document:

<h1 align="center">LoR Showcase</h1>

**Status:** Ready

**Developers:** <a href="https://github.com/xtecna">Raquel "xTecna" Marcolino</a> (Backend/DevOps) and <a href="https://github.com/renatocesarramos">Renato "AloRenato" Ramos</a> (Frontend)

**Key Collaborator:** <a href="https://www.twitch.tv/viktorkav">Victor "ViktorKav" Cavalcante</a>

This project aims to display LoR Spoiler Cards in a timely fashion, with the best practices to keep it secure and fast. It was also designed to display concise information about the current reveal progress, how many cards are next and when they will be revealed and a nice gallery with filters by region and options to zoom out cards and card arts.

<h3 align="center">Visit our page at <a href="https://escolaruneterra.com.br/guardioes-dos-ancestrais/">https://escolaruneterra.com.br/guardioes-dos-ancestrais/</a></h3>

<hr/>

<h2>Digital Ocean</h2>

<h3>Structure</h3>

We use Digital Ocean Droplet services in order to maintain control of our backend and database platform. Our droplet has the following specifications:

**vCPUs:** 1 vCPU

**Memory:** 1 GB

**SSD:** 25 GB Disk

**Transfer data:** 1 TB

**Server location:** New York City

**Operational system:** Ubuntu 20.04 (LTS) x64

<h3>Security Measures</h3>

Our application is running through NGINX and can only be accessed by the URL [https://api.escolaruneterra.com.br/](https://api.escolaruneterra.com.br/). Doing so will return a Hello World message whose purpose is to inform that everything is OK within the server.

To improve our server's security, all authentications must be made with a SSH key instead of a password and the user *root* is disabled. There is also a firewall that only allows remote/external access to the server on the API port (3333), HTTP and HTTPS ports, keeping closed the port to the database.

It is worth of note that our API has a SSL signature, which ensures that every request will be performed on HTTPS, a more secure protocol than HTTP.

<hr/>

<h2>PostgreSQL</h2>

<h3>Structure</h3>

We are using PostgreSQL as our database, which only has three tables:

**knex_migrations** and **knex_migrations_lock**: They're mainly used for Knex, this project's ORM.

**cards**: The main table, used to store card information.

<table>
  <tr><th>Column Name</th><th>Type</th><th>NULL?</th><th>Description</th></tr>
  <tr><td>id</td><td>VARCHAR</td><td>❌</td><td>Card ID</td></tr>
  <tr><td>revealed</td><td>DATETIME WITH TIMEZONE</td><td>❌</td><td>When this card will be revealed to the public</td></tr>
  <tr><td>category</td><td>VARCHAR</td><td>❌</td><td>Which set this card is in</td></tr>
</table>

When we save a new card to the database, we enter the reveal date with PST timezone and it is automatically converted to the server current timezone.

At the Node.js section, we can see that all our queries are filtered through the *category* column, so it's only natural that this table is indexed by this column to improve performance.

<h3>Security Measures</h3>

To avoid leaking cards before the due time, we attach a random sequence of letters and numbers at the end of the category name. These are the two categories present in our database until this present moment:

- empires-of-the-ascended-Phvte2Q3d5wFfHU
- guardians-of-the-ancient-RFYCCPp8pHwp4VB

Note that when the card gallery is on-line and open to the public access, the category name is revealed to everyone, as its name is necessary to access the available cards. At Node.js section, we presented which pieces of information will become available once the name is revealed and everyone can use it to consult the API.

We also have a specific user to access this database, deactivating the standard *postgres* user.

<hr/>

<h2>Node.js</h2>

<h3>Structure</h3>

Our API was written in Node.js with Express as our framework of choice and Knex as our chosen ORM. It is quite simple as it only has two routes.

<h4>/ route</h4>

**Possible Responses:**

**Status 200**
Returns a *Hello World* message to assure that the service is up and running.

<h4>/cards route</h4>

**Required query:** A "cat" query relating to the set of cards the page intends to display.

Example: [http://api.escolaruneterra.com.br/cards?cat=guardians-of-the-ancient-RFYCCPp8pHwp4VB](http://api.escolaruneterra.com.br/cards?cat=guardians-of-the-ancient-RFYCCPp8pHwp4VB)

**Possible Responses:**

**Status 400 — Bad Request**
If there isn't a specified query in the URL or if there isn't a "cat" query.

**Status 503 — Service Unavailable**
If the database isn't available at the time of the request.

**Status 200**
Returns a JSON structure as the response body. Note that this structure is returned even if the passed category is invalid, although with empty fields.

**Response Structure:**

```json
{
	cards:
		[
			{
				region: "shurima"
				url: "http://dd.b.pvp.net/latest/set4/pt_br/img/cards/04SH003.png"
				url_full: "http://dd.b.pvp.net/latest/set4/pt_br/img/cards/04SH003-full.png"
			},
			...
		],
	numbers:
		{
			revealed: 165,
			total: 170,
		},
	future:
		{
			to_be_revealed: 5,
			diff: 108000,
		},
}
```

- The section *cards* has a list of every card revealed until the moment.
    - Each card has a region, an image URL and a full art URL.
- The section *numbers* has information on the number of cards already revealed and of the cards that will be revealed until the end of the spoilers season.
- The section *future* has information on the number of cards to be revealed in the next batch of spoilers and the difference of time in seconds between that moment and the current moment of the request in server time.
    - As 7 days of difference is 604800 seconds, we deemed safe saving this number in an integer variable.

<h3>ORM Structure</h3>

These are the queries we use to gather each information presented by the */cards* route.

<h4>Cards</h4>

```jsx
let cards = await connection("cards")
	.select("id")
  .where("category", "=", category)
  .andWhere("revealed", "<=", now)
  .orderBy("revealed", "desc")
  .orderBy("id", "asc");
cards = cards.map((item) => {
	const id = item.id;
	  return {
	    region: getRegion(id),
	    url: getUrl(id, false),
      url_full: getUrl(id, true),
    };
  }
);
```

Being *category* the category on the query and *now* the present moment in server time.

<h4>Numbers</h4>

```jsx
const numbers = {
	revealed: cards.length,
  total: +(
	  await connection("cards").count("id").where("category", "=", category)
  )[0]["count"],
};
```

<h4>Future</h4>

```jsx
const nextDeadline = await connection("cards")
  .select("revealed")
  .where("category", "=", category)
  .andWhere("revealed", ">", now)
  .orderBy("revealed")
  .first();
const to_be_revealed = nextDeadline ?
  +(await connection("cards")
	  .count("id")
    .where("revealed", "=", nextDeadline.revealed)
  )[0]["count"] : 0;
const diff = nextDeadline ? getDiffSeconds(nextDeadline.revealed, now) : -1;
const future = {
	to_be_revealed,
  diff,
};
```

Here we can see that if there are no more cards to be revealed, *diff* assumes value -1.

<h3>Security Measures</h3>

The *now* variable represents the time in **server time**. This ensures that the time cannot be manipulated on the client side, so the countdown will not increase or decrease based on local timezone. This will also guarantee that no card will be available before the server reaches the due time, according to its own timezone.

<hr/>

<h2>JavaScript</h2>

Viktor wrote the site in WordPress and therefore, we decided on using only CSS and JavaScript in the frontend for maximum compatibility. Our page is also mobile friendly!

<h3>Structure</h3>

Each LoR Showcase page imports the required *script.js* like this:

```html
<script>const pageTitle = "Guardiões Ancestrais";
const pageSet = "guardians-of-the-ancient-RFYCCPp8pHwp4VB";</script>
<script src="/gallerycard/script.js">
```

Where *pageTitle* and *pageSet* refer to a specific page settings (in other words, which set will be presented in this page).

Then, we fetch a request from the API as follows:

```jsx
async function getData(currentSet){
  renderLoading(true);
    
  const response = await fetch(`https://api.escolaruneterra.com.br/cards?cat=${currentSet}`);

  if (response.ok){
    const result = await response.json();
    data = result;
        
    renderProgressBar(data.numbers.revealed, data.numbers.total);
    renderCountdown(data.future.to_be_revealed, data.future.diff);
    renderCards(data.cards);
    renderLoading(false);
  }else{
    document.getElementById('progress_bar').style.display = 'none';
    document.getElementById('next_spoiler').style.display = 'none';
    document.getElementById('loading').innerHTML = '<h3>Ocorreu um erro. Por favor, recarregue a página.</h3>';
  }
}
```

Where *currentSet* is the *pageSet* mentioned above.

<h3>Security Measures</h3>

The API call is the only way we can communicate with the backend and only once, when the page loads. Once the page loads and the cards appear, they're saved in memory and can be filtered through the filter interface, that works exactly the same as the Legends of Runeterra one.
