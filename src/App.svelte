<script>
  import Form from "./Form.svelte";
  import { onMount } from "svelte";
  import csvData from "./data.js";

  // score: Your ACT
  // gpa: Your GPA
  // score_[50|75|25]_perc: School's ACT distribution, 75 is lowest ACT, 25 will be highest
  // average_gpa: School's average GPA
  // admissions rate: Admissions percentage
  function chances(
    student_act,
    student_gpa,
    school_50_act,
    school_75_act,
    school_25_act,
    school_gpa,
    school_admission_rate,
  ) {
    let gpa_calc;
    if ([null, undefined, ""].includes(school_gpa)) {
      school_gpa = NaN;
    }
    if (!isNaN(school_gpa) && student_gpa != undefined) {
      gpa_calc = (student_gpa - school_gpa) / 0.25;
    } else {
      gpa_calc = 0;
    }
    let score_exp =
      (student_act - school_50_act) / ((school_75_act - school_25_act) / 2);
    let y =
      (school_admission_rate / 100.0) *
      Math.pow(2, score_exp) *
      Math.pow(2, gpa_calc);
    let chance =
      ((100 * y) / (100 * y + 100.0 - school_admission_rate)) * 100.0;
    if (school_admission_rate == 100) {
      chance = 100;
    }
    chance = parseInt(chance * 100) / 100.0;
    if (chance > 100) chance = 100;
    return chance;
  }

  let data = {};

  $: act_ranges = data.school_act
    ?.split("-")
    .map((i) => i.trim())
    .map((i) => parseInt(i))
    .sort((a, b) => a - b) || [0, 0];

  $: chance = chances(
    data.act,
    data.gpa,
    // Assume 50% is directly in middle
    (act_ranges[0] + act_ranges[1]) / 2,
    act_ranges[1],
    act_ranges[0],
    data.school_gpa,
    data.school_acceptance_rate,
  );

  let setValue;

  let colleges = [];

  onMount(() => {
    // Svelte has a "loop guard" that prevents this from workin
    eval(
      JSON.parse(
        `"window.parseData=function parse(r,n){n=n||function(r,n,e){return e};for(var e,f,i,o=r.split(\\"\\"),t=0,l=o.length,s=[];t<l;){for(s.push(i=[]);t<l&&\\"\\\\r\\"!==o[t]&&\\"\\\\n\\"!==o[t];){if(e=f=t,'\\"'===o[t]){for(e=f=++t;t<l;){if('\\"'===o[t]){if('\\"'!==o[t+1])break;o[++t]=\\"\\"}f=++t}for('\\"'===o[t]&&++t;t<l&&\\"\\\\r\\"!==o[t]&&\\"\\\\n\\"!==o[t]&&\\",\\"!==o[t];)++t}else for(;t<l&&\\"\\\\r\\"!==o[t]&&\\"\\\\n\\"!==o[t]&&\\",\\"!==o[t];)f=++t;i.push(n(s.length-1,i.length,o.slice(e,f).join(\\"\\"))),\\",\\"===o[t]&&++t}\\"\\\\r\\"===o[t]&&++t,\\"\\\\n\\"===o[t]&&++t}return s}"`,
      ),
    );

    const isNumeric = (num) =>
      (typeof num === "number" ||
        (typeof num === "string" && num.trim() !== "")) &&
      !isNaN(num);
    const lines = parseData(csvData.trim());
    let header = lines[0];
    let data = lines.slice(1);

    for (let item of data) {
      colleges.push(
        Object.fromEntries(item.map((i, idx) => [header[idx], parse(i)])),
      );
    }
    colleges = [...colleges].filter(
      (i) =>
        i["ACT Composite 25th percentile score"] &&
        i["ACT Composite 75th percentile score"] &&
        i["Percent admitted - total"],
    );

    function parse(i) {
      if (isNumeric(i)) {
        return parseFloat(i, 10);
      }
      if (!i.length) {
        return null;
      }
      return i;
    }
  });

  function alphaSort(arr, key) {
    return arr.sort(function (a, b) {
      return a[key].localeCompare(b[key]);
    });
  }

  function handle() {
    const sel = document.querySelector("#college_opts");
    const val = sel.options[sel.selectedIndex].value;
    setValue(getInfo(val));
  }

  const ACT_TO_GPA = `36	4.00
35	4.00
34	3.99
33	3.99
32	3.99
31	3.98
30	3.98
29	3.97
28	3.95
27	3.93
26	3.89
25	3.84
24	3.76
23	3.67
22	3.54
21	3.39
20	3.23
19	3.05
18	2.85
17	2.64
16	2.37
15	2.05
14	1.65
13	0.81
12	0.00
11	0.00
10	0.00
9	0.00
8	0.00
7	0.00
6	0.00
5	0.00
4	0.00
3	0.00
2	0.00
1	0.00`
    .split("\n")
    .map((i) => i.split("\t").filter((j) => j.trim().length))
    .map((i) => i.map((j) => parseFloat(j, 10)));

  function getGPA(act) {
    return ACT_TO_GPA.find((i) => i[0] == act)[1];
  }
  function getInfo(val) {
    const college = colleges.find((i) => i["Name"] == val);
    const avg_act =
      (parseInt(college["ACT Composite 25th percentile score"]) +
        parseInt(college["ACT Composite 75th percentile score"])) /
      2;
    return {
      school_act: `${college["ACT Composite 25th percentile score"]}-${college["ACT Composite 75th percentile score"]}`,
      school_acceptance_rate: college["Percent admitted - total"],
      school_gpa: isNaN(avg_act) ? "" : getGPA(Math.round(avg_act)),
    };
  }
  onMount(() => {
    setTimeout(() => {
      setVal("Harvard University");
    });
  });

  function setVal(v) {
    let sel = document.querySelector("#college_opts");
    sel.value = v;
    setTimeout(handle);
  }

  function getChance(collegeName, data) {
    const college = getInfo(collegeName);
    let act_ranges = college.school_act
      ?.split("-")
      .map((i) => i.trim())
      .map((i) => parseInt(i))
      .sort((a, b) => a - b) || [0, 0];
    return chances(
      data.act,
      data.gpa,
      // Assume 50% is directly in middle
      (act_ranges[0] + act_ranges[1]) / 2,
      act_ranges[1],
      act_ranges[0],
      college.school_gpa,
      college.school_acceptance_rate,
    );
  }

  function padTo(str, len) {
    return str.slice(0, len - 5).padEnd(len, "-");
  }

  let showColleges = "all";

  function filterColleges(colleges, level = "all") {
    let curr = document.querySelector("#college_opts")?.value;
    level = level?.toLowerCase()?.trim();
    if (level === "all") {
      return colleges;
    }
    if (!(data.act && data.gpa)) {
      return colleges;
    }
    let out = colleges
      .map((c) => {
        const myChance = getChance(c["Name"], data);
        return {
          ...c,
          chance: myChance < 20 ? "reach" : myChance < 70 ? "match" : "safety",
        };
      })
      .filter((i) => i.chance === level);
    if (!out?.length) {
      alert("No colleges are a " + level + " for you");
      showColleges = "all";
    }
    if (!out.find((i) => i["Name"] === curr)) {
      setTimeout(() => setVal(out[0]["Name"]));
    } else {
      setVal(curr);
    }
    return out;
  }
</script>

<div class="outer">
  <Form
    class="form_container"
    type="form"
    bind:setValue
    on:update={(a) => (data = a.detail)}
  >
    <h2>About you</h2>
    <Form
      type="slider"
      min={0}
      max={5}
      step={0.1}
      description="Weighted GPA"
      value={4}
      id="gpa"
      label="Your GPA"
    />
    <Form
      type="slider"
      min={0}
      max={36}
      step={1}
      description="Your ACT score"
      id="act"
      value={33}
      label="ACT Score"
    />

    <h2>The college</h2>
    <span>Show:</span>
    <select id="show_colleges" bind:value={showColleges}>
      <option value="all">All</option>
      <option value="safety">Safety</option>
      <option value="match">Match</option>
      <option value="reach">Reach</option>
    </select>
    <select id="college_opts" on:change={handle}>
      {#each filterColleges(alphaSort(colleges, "Name"), showColleges) as c}
        <option value={c["Name"]}
          >{padTo(c["Name"], 25)}---{getChance(c["Name"], data)}% chance</option
        >
      {/each}
    </select>
    <i style="display: block; color: #666; margin: 5px;"
      >You can also put custom stats below</i
    >
    <Form
      type="number"
      value="3.9"
      id="school_gpa"
      placeholder="GPA"
      label="School's GPA"
      description="The average GPA for the school in question"
    />
    <Form
      type="input"
      id="school_act"
      placeholder="e.g. 29-24"
      value="31-34"
      validate={({ value }) =>
        value.split("-").length === 2 &&
        value
          .split("-")
          .map((i) => i.trim())
          .every((i) => parseInt(i) < 36 && parseInt(i) > 0)
          ? true
          : "Must be range with both between 0 and 36"}
      label="School's ACT distribution"
      description="Distribution of ACT scores, enter a range that includes 75% of students"
    />
    <Form
      type="number"
      id="school_acceptance_rate"
      description="The acceptance rate of your school by percentage"
      label="Acceptance rate"
      placeholder="__%"
      value="6.7"
    />
  </Form>
  <div class="score">
    <span class="desc">Your change of getting in:</span>
    <div class="score">{chance}%</div>
    <span class="analysis"
      >{#if chance < 5}
        You probably won't get in 🖕
      {:else if chance < 20}
        Reach school ✋
      {:else if chance < 65}
        You might get in, you might not 🤷
      {:else if chance < 80}
        You'll probably get in 🎉
      {:else}
        Safety school! 🔒
      {/if}
    </span>
    <span class="credits"
      >Made by <a href="https://github.com/explosion-scratch"
        >Explosion-Scratch</a
      >. All processing is done in your browser, your GPA is safe =)</span
    >
  </div>
</div>
<svelte:head>
  <style>
    html {
      line-height: 1.15;
      -webkit-text-size-adjust: 100%;
    }
    body {
      margin: 0;
    }
    main {
      display: block;
    }
    h1 {
      font-size: 2em;
      margin: 0.67em 0;
    }
    hr {
      box-sizing: content-box;
      height: 0;
      overflow: visible;
    }
    pre {
      font-family: monospace, monospace;
      font-size: 1em;
    }
    a {
      background-color: transparent;
    }
    abbr[title] {
      border-bottom: none;
      text-decoration: underline;
      -webkit-text-decoration: underline dotted;
      text-decoration: underline dotted;
    }
    b,
    strong {
      font-weight: bolder;
    }
    code,
    kbd,
    samp {
      font-family: monospace, monospace;
      font-size: 1em;
    }
    small {
      font-size: 80%;
    }
    sub,
    sup {
      font-size: 75%;
      line-height: 0;
      position: relative;
      vertical-align: baseline;
    }
    sub {
      bottom: -0.25em;
    }
    sup {
      top: -0.5em;
    }
    img {
      border-style: none;
    }
    button,
    input,
    optgroup,
    select,
    textarea {
      font-family: inherit;
      font-size: 100%;
      line-height: 1.15;
      margin: 0;
    }
    button,
    input {
      overflow: visible;
    }
    button,
    select {
      text-transform: none;
    }
    [type="button"],
    [type="reset"],
    [type="submit"],
    button {
      -webkit-appearance: button;
    }
    [type="button"]::-moz-focus-inner,
    [type="reset"]::-moz-focus-inner,
    [type="submit"]::-moz-focus-inner,
    button::-moz-focus-inner {
      border-style: none;
      padding: 0;
    }
    [type="button"]:-moz-focusring,
    [type="reset"]:-moz-focusring,
    [type="submit"]:-moz-focusring,
    button:-moz-focusring {
      outline: 1px dotted ButtonText;
    }
    fieldset {
      padding: 0.35em 0.75em 0.625em;
    }
    legend {
      box-sizing: border-box;
      color: inherit;
      display: table;
      max-width: 100%;
      padding: 0;
      white-space: normal;
    }
    progress {
      vertical-align: baseline;
    }
    textarea {
      overflow: auto;
    }
    [type="checkbox"],
    [type="radio"] {
      box-sizing: border-box;
      padding: 0;
    }
    [type="number"]::-webkit-inner-spin-button,
    [type="number"]::-webkit-outer-spin-button {
      height: auto;
    }
    [type="search"] {
      -webkit-appearance: textfield;
      outline-offset: -2px;
    }
    [type="search"]::-webkit-search-decoration {
      -webkit-appearance: none;
    }
    ::-webkit-file-upload-button {
      -webkit-appearance: button;
      font: inherit;
    }
    details {
      display: block;
    }
    summary {
      display: list-item;
    }
    template {
      display: none;
    }
    [hidden] {
      display: none;
    }
    html {
      font-family: sans-serif;
    }
    .hidden,
    [hidden] {
      display: none !important;
    }
    .pure-img {
      max-width: 100%;
      height: auto;
      display: block;
    }
  </style>
</svelte:head>

<style>
  :global(*) {
    box-sizing: border-box;
	font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
  }
  :global(body){
	display: flex;
	justify-content: center;
	align-items: center;
	min-height: 100vh;
  }
  .outer {
    display: flex;
    max-width: 800px;
	margin: 10px auto;
    gap: 20px;
	position: fixed;
	inset: 20px;
  }
  h2 {
	margin-top: 0;
  }
  :global(form) {
    flex: 1;
    overflow-y: scroll;
  }
  select {
	border: 1px solid #eee;
	padding: 2px;
	margin: 3px;
  }
  .outer > * {
    height: 100%;
  }
  .score {
    font-size: 1.2em;
    width: 300px;
	flex: 1;
	position: relative;
  }
  .score .desc {
    font-size: 1em;
    font-style: italic;
    color: #666;
    font-weight: 300;
  }
  .score .score {
    font-size: 4em;
    font-weight: 100;
  }
  .credits {
    position: absolute;
    bottom: 30px;
    width: 100%;
    right: 0;
    left: 0;
    font-size: 0.6em;
  }
  .analysis {
    color: #007a8c;
  }
</style>
