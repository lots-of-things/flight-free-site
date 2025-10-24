---
layout: page
title: Flight Emissions Calculator
permalink: /calculator/
---

<link
  href="https://cdn.jsdelivr.net/npm/select2@4.0.13/dist/css/select2.min.css"
  rel="stylesheet"
/>
<script src="https://cdn.jsdelivr.net/npm/jquery@3.5.0/dist/jquery.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/select2@4.0.13/dist/js/select2.min.js"></script>

<script type="text/javascript">
  // TODO:
  // - Select round-trip or not
  // - Add more flights? (lower priority)
  // - Fix the flight emissions conversion factor
  // - Add sources
  // - Add example flights, most common routes
  // - Add percentage of carbon budget

  const MILES_PER_KM = 0.6213712;
  const POUNDS_PER_KG = 2.204623;
  const SQ_FT_PER_SQ_M = 10.76391;

  let data = null;
  let departure = null;
  let arrival = null;
  let ghgCdf = null;
  let initialized = false;

  let $departureSelect = null;
  let $arrivalSelect = null;

  function recompute() {
    if (departure === null || arrival === null) return;

    initialized = true;

    const earthRadius = 6378.137; // kilometers
    let dLat = ((arrival.lat - departure.lat) * Math.PI) / 180;
    let dLon = ((arrival.long - departure.long) * Math.PI) / 180;
    let a =
      Math.pow(Math.sin(dLat / 2), 2) +
      Math.pow(Math.sin(dLon / 2), 2) *
        Math.cos((departure.lat * Math.PI) / 180) *
        Math.cos((arrival.lat * Math.PI) / 180);
    let distance = 2 * earthRadius * Math.asin(Math.sqrt(a));

    $("#distance").text(
      (distance * MILES_PER_KM).toLocaleString(undefined, {
        maximumFractionDigits: 0,
      }) +
        " miles or " +
        distance.toLocaleString(undefined, { maximumFractionDigits: 0 }) +
        " km",
    );

    // 88g/km and 1.9x multplier is pretty consensus. Plus 8% for indirect flights and delays.
    const kgCo2PerKm = 0.088 * 1.9 * 1.08;
    let emissions = 2 * distance * kgCo2PerKm;
    $("#emissions").text(
      (emissions / 1000).toLocaleString(undefined, {
        maximumFractionDigits: 1,
      }) + " metric tons CO2 equivalent",
    );

    const vegReductionPerYear = 540.0; // kilograms
    let veg = emissions / vegReductionPerYear;
    $("#veg").text(
      veg.toLocaleString(undefined, { maximumFractionDigits: 1 }) + " years",
    );

    const carpoolReductionPerYear = 1017.0; // kilograms
    let carpool = emissions / carpoolReductionPerYear;
    $("#carpool").text(
      carpool.toLocaleString(undefined, { maximumFractionDigits: 1 }) +
        " years",
    );

    if (ghgCdf !== null) {
      let threshold = ghgCdf.find((e) => e[0] > emissions / 1000);
      let pop = threshold[1];
      if (pop < 1e9) {
        $("#pop").text(
          (pop / 1e6).toLocaleString(undefined, { maximumFractionDigits: 0 }) +
            " million",
        );
      } else {
        $("#pop").text(
          (pop / 1e9).toLocaleString(undefined, { maximumFractionDigits: 1 }) +
            " billion",
        );
      }
    }

    const iceFactor = 0.003; // 3 sq meters per ton of CO2
    let seaice = emissions * iceFactor;
    $("#seaice").text(
      (seaice * SQ_FT_PER_SQ_M).toLocaleString(undefined, {
        maximumFractionDigits: 1,
      }) +
        " square feet or " +
        seaice.toLocaleString(undefined, { maximumFractionDigits: 1 }) +
        " square meters",
    );

    const earthCircum = 40e3; // 40,000km
    const eurostarEmissions = 0.006; // kg per pkm (6 grams)
    let eurostarDist = emissions / eurostarEmissions;
    let timesAroundWorld = eurostarDist / earthCircum;
    $("#eurostar").text(
      timesAroundWorld.toLocaleString(undefined, { maximumFractionDigits: 1 }) +
        " times around the world",
    );
  }

  function setAirports(iata1, iata2) {
    let airport1 = data.find((e) => e.id === iata1);
    let airport2 = data.find((e) => e.id === iata2);
    $departureSelect
      .val(iata1)
      .trigger("change")
      .trigger({ type: "select2:select", params: { data: airport1 } });
    $arrivalSelect
      .val(iata2)
      .trigger("change")
      .trigger({ type: "select2:select", params: { data: airport2 } });
  }

  // Note: the name of this function is hardcoded into the JS file due to JSONP
  // Must use JSONP due to CORS and because Squarespace static assets get redirected
  // from flightfreeusa.org to squarespace.com, so we can't use for storing static assets.
  function jsonpCallback(d) {
    data = d;
    data.forEach((element) => {
      element.text = element.id + " - " + element.name;
    });

    $departureSelect = $("#departure");
    $arrivalSelect = $("#arrival");

    $departureSelect.select2({
      data: data,
      placeholder: "Departure Airport (e.g., SFO)",
    });

    $arrivalSelect.select2({
      data: data,
      placeholder: "Arrival Airport (e.g., JFK)",
    });

    $departureSelect.on("select2:select", function (e) {
      departure = e.params.data;
      recompute();
    });

    $arrivalSelect.on("select2:select", function (e) {
      arrival = e.params.data;
      recompute();
    });

    // Start with some defaults.
    setAirports("SFO", "JFK");
  }

  // Note: The name of this function is hardcoded in the JSONP
  function ghgCallback(d) {
    ghgCdf = d;
    // If we've already started computing, then refresh so that the GHG data shows up.
    if (initialized) recompute();
  }

  $(function () {
    $("#departure").select2({ placeholder: "Loading..." });
    $("#arrival").select2({ placeholder: "Loading..." });
    const url =
      "https://cdn.jsdelivr.net/gh/thenovices/flight-calculator/data/data-jsonp.js";
    $.getJSON(url + "?callback=?");
    const ghgUrl =
      "https://cdn.jsdelivr.net/gh/thenovices/flight-calculator/data/emissions-cdf-jsonp.js";
    $.getJSON(ghgUrl + "?callback=?");
  });
</script>

<style>
  #calculator {
    border: 1px solid #333;
    padding: 20px;
    font-size: 18px;
    display: flex;
    flex-direction: column;
    border-radius: 8px;
    background: rgba(255, 255, 255, 0.05);
    margin-bottom: 30px;
  }

  #airports {
    text-align: center;
    width: 100%;
    display: flex;
    margin-bottom: 20px;
  }

  #airports > * {
    width: 50%;
    padding: 0 10px;
  }

  @media (max-width: 600px) {
    #airports {
      flex-direction: column;
    }

    #airports > * {
      width: 100%;
      margin-top: 10px;
      padding: 0;
    }
  }

  #calculator ul {
    list-style: none;
    padding-left: 0;
  }

  #calculator li {
    margin-top: 15px;
    line-height: 1.6;
  }

  #calculator li span {
    font-weight: bold;
    color: #0066cc;
  }

  a.sources {
    align-self: flex-end;
    margin-top: 20px;
  }
</style>

<div id="calculator">
  <div id="airports">
    <div>
      <div><strong>Departure</strong></div>
      <select
        id="departure"
        class="airport-select"
        name="departure"
        style="width: 90%"
      >
        <option></option>
      </select>
    </div>
    <div>
      <div><strong>Arrival</strong></div>
      <select
        id="arrival"
        class="airport-select"
        name="arrival"
        style="width: 90%"
      >
        <option></option>
      </select>
    </div>
  </div>

  <ul>
    <li>Distance (each way): <span id="distance">N/A</span></li>
    <li>Round-trip emissions per passenger: <span id="emissions">N/A</span></li>
    <li>
      Avoiding this trip is as climate friendly as being vegetarian for:
      <span id="veg">N/A</span>
    </li>
    <li>
      Avoiding this trip is as climate friendly as carpooling for:
      <span id="carpool">N/A</span>
    </li>
    <li>
      This many people in the world emit fewer greenhouse gases in one year:
      <span id="pop">N/A</span>
    </li>
    <li>
      You could travel this far in an electric train like Eurostar:
      <span id="eurostar">N/A</span>
    </li>
    <li>
      These emissions melt this much Arctic sea ice:
      <span id="seaice">N/A</span>
    </li>
  </ul>

  <a class="sources" href="https://github.com/branliu0/flight-calculator"
    >code & sources</a
  >
</div>


