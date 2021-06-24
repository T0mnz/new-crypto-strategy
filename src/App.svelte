<script>
import { onMount } from "svelte";
import dayjs from "dayjs";
import Decimal from "decimal.js";
import buildPairs from "./lib/pairs";
import buildProfitFunc from "./lib/profit";

import socketSubscribe from "./lib/socket";

import ResultsTable from "./winners-table.svelte";
import DetailModal from "./detail.svelte";

let modalDetail = false;
let contentModal = {};

const showPopupLong = e => {

  modalDetail = true;
  contentModal = e;

};

const info = {
  pairslen: 0,
  msgcount: 0,
  cycleschecked: 0,
  cyclescheckedPerSecond: 0,
  currentStatus: "",
}
let oldWinners = null;
let winners = [];
let tops = [];

let checkProfit;

onMount( async () => {

  const socket = await socketSubscribe();
  const { hashMarket, pairs: allPairs } = await buildPairs();

  info.currentStatus = "Connecting to binance";

  info.pairslen = allPairs.length;
  info.socket = socket;
  setInterval( () => {

    info.msgcount = socket.msgcount;
    socket.msgcount = 0;
    info.cyclescheckedPerSecond = info.cycleschecked;
    info.cycleschecked = 0;

  }, 1000 )

  checkProfit = await buildProfitFunc();

  function tick() {

    if ( info.currentStatus === "Connecting to binance" ) {

      if ( info.msgcount > 0 ) {

        info.currentStatus = "Working";

      }

    } else {

      info.currentStatus = "Working";

    }

    const startTime = +new Date();
    const uPair = socket.pairupdate;
    const markets = socket.m;

    socket.pairupdate = [];

    let pairsToTest = uPair.map( pair => hashMarket[ pair ] ).flat()
    pairsToTest = [ ...new Set( pairsToTest ) ].map( id => allPairs[ id ] ).filter( e => e );

    if ( ! pairsToTest.length ) {

      setTimeout( tick, 50 );
      return;

    }

    // voy con todos!
    // const pairsToTest = allPairs.filter(p => intersect(p, u).length)

    let ret;
    try {

      info.cycleschecked += pairsToTest.length;
      ret = checkProfit( pairsToTest, markets );
      ret.sort( ( a, b ) => b.profit.sub( a.profit ) )

      const w = ret.filter( e => e.profit.gt( 1 ) );
      if ( w.length ) {

        winners = w;
        console.log( winners );
        w.forEach( e => {

          e.time = dayjs().format();

          for ( let k = 0; k < ( e.length - 1 ); k += 1 ) {

            delete socket.m[ e[ k ] + "/" + e[ k + 1 ] ];
            delete socket.m[ e[ k + 1 ] + "/" + e[ k ] ];

          }

        } );

      }

    } catch ( err ) {

      console.error( err )
      console.log( pairsToTest )

    }

    console.log( "cycle", +new Date() - startTime, "ms" )
    if ( winners.length ) {

      info.currentStatus = "Found profitable cycles, halt for 30 seconds";
      // halt 30 seconds
      setTimeout( () => {

        info.currentStatus = "Working";
        oldWinners = winners;
        winners = [];
        tick();

      }, 1000 * 30 )
      return;

    }

    tops = ret
      .slice( 0, 10 );

    setTimeout( tick, 50 )

  }
  setTimeout( tick, 1000 );
  return () => {}

} );
</script>
<nav class="navbar navbar-expand-md navbar-dark bg-dark fixed-top">
  <a class="navbar-brand" href="#">Navbar</a>
  <button class="navbar-toggler" type="button" data-toggle="collapse" data-target="#navbarsExampleDefault" aria-controls="navbarsExampleDefault" aria-expanded="false" aria-label="Toggle navigation">
    <span class="navbar-toggler-icon"></span>
  </button>

  <div class="collapse navbar-collapse" id="navbarsExampleDefault">
    <ul class="navbar-nav mr-auto">
      <li class="nav-item active">
        <a class="nav-link" href="#">Home <span class="sr-only">(current)</span></a>
      </li>
      <li class="nav-item">
        <a class="nav-link" href="#">Link</a>
      </li>
      <li class="nav-item">
        <a class="nav-link disabled" href="#" tabindex="-1" aria-disabled="true">Disabled</a>
      </li>
      <li class="nav-item dropdown">
        <a class="nav-link dropdown-toggle" href="#" id="dropdown01" data-toggle="dropdown" aria-haspopup="true" aria-expanded="false">Dropdown</a>
        <div class="dropdown-menu" aria-labelledby="dropdown01">
          <a class="dropdown-item" href="#">Action</a>
          <a class="dropdown-item" href="#">Another action</a>
          <a class="dropdown-item" href="#">Something else here</a>
        </div>
      </li>
    </ul>
    <form class="form-inline my-2 my-lg-0">
      <input class="form-control mr-sm-2" type="text" placeholder="Search" aria-label="Search">
      <button class="btn btn-secondary my-2 my-sm-0" type="submit">Search</button>
    </form>
  </div>
</nav>
<main>
	
	Total combinations: {info.pairslen} pairs<br >
	Socket updates per second: {info.msgcount}<br />
	Cycles checked per second: {info.cyclescheckedPerSecond}<br />

	<big>
		Current status<br>
		<b>{info.currentStatus}</b><br />
	</big>


	
	<h2>WINNERS</h2>
	{#if ! winners.length}
		<b>No winner pair yet</b>
	{:else}
		<ResultsTable results={winners} />
	{/if}


	{#if tops.length}
	<h2>Pairs</h2>
	<table class="styled-table pairs-table">
		<thead>
			<th>Pairs cycle</th>
			<th>Profit (BNB Fess included)</th>
		</thead>
		<tbody>
			{#each tops as t}
			<tr>
				<td style="min-width: 235px;">
					{@html t.chain.join( " &rarr; " )}
				</td>
				<td>
					<pre style="cursor:pointer" on:click={() => showPopupLong( t )}>{t.profit.sub(1).mul(100).toFixed( 4 )}% &#9432;</pre>
				</td>
				</tr>
			{/each}
		</tbody>
	</table>
	{/if}
	<blockquote>
		Trades profit are calculate on a base budget of 100USD and a 0.0075% fee
	</blockquote>

	{#if oldWinners && oldWinners.length}
		<h2>PREV CYCLE WINNERS</h2>
		<ResultsTable results={oldWinners} />
	{/if}

</main>
{#if modalDetail}
	<DetailModal on:close={() => modalDetail = false}>
		<ResultsTable results={[ contentModal ]} />
	</DetailModal>
{/if}

<style>
	table {
		margin: auto;
	}
	main {
		text-align: center;
		padding: 1em;
		max-width: 240px;
		margin: 0 auto;
	}

	h1 {
		color: #ff3e00;
		text-transform: uppercase;
		font-size: 2em;
		font-weight: 400;
	}

	@media (min-width: 640px) {
		main {
			max-width: none;
		}
	}

  .pairs-table {
    max-width: 320px; margin: 30px auto;
  }
  .pairs-table td {
    padding: 5px 15px;
  }

</style>
