<script>
	function cMake(mag, phase) {
		return {mag, phase}
	}
	
	var seed = 1;
	function random() {
			var x = Math.sin(seed++) * 10000;
			return x - Math.floor(x);
	}
	
	let signalLengthExp = 6
	$: signalLength = Math.pow(2, signalLengthExp)
	$: signal = Array(signalLength).fill(null).map((_,i) => cMake(random(), Math.sin(i/37*2*Math.PI)*2*Math.PI))
	$: maxMag = signal.reduce((acc, {mag}) => Math.max(Math.abs(mag), acc),0)
	
	let filterShape = 'rect'

	let oddLength = false
	$: oddLength = (windowLengthExp==0 || windowLengthExp == signalLengthExp) ? false : oddLength
	let windowLengthExp = 3
	$: windowLengthExp = Math.min(windowLengthExp, signalLengthExp)
	$: windowLength = Math.pow(2, windowLengthExp) + (oddLength?1:0)

	let autoShiftSize = false
	let maxShiftSize = false
	$: windowShiftSize = 1
	$: windowShiftSize = autoShiftSize ? maxShiftSize ? windowLength : Math.ceil(windowLength / 2) : Math.min(windowShiftSize, windowLength)
	
	let filterStep = 0
	$: filterStepMax = Math.floor((signalLength-windowLength) / windowShiftSize)
	$: filterStep = Math.min(filterStep, filterStepMax)
	
	$: omittedCount = signalLength - (filterStepMax * windowShiftSize + windowLength)
	
	let autoFrequency = false
	let filterFrequency = 0
	$: maxFilterFrequency = Math.pow(2, windowLengthExp)-1
	$: filterFrequency = autoFrequency ? Math.min(windowLength-1, 1) : Math.min(filterFrequency, maxFilterFrequency)
	$: filterWindow = Array(windowLength).fill(null).map(
		(_,i) => cMake(
			filterShape == 'rect' ? 1 :
			Math.exp(-Math.pow(((i+0.5)/(windowLength+1)-(0.5))*4, 2)), 
			filterFrequency*i/(windowLength)*2*Math.PI))

	$: allSteps = Array(filterStepMax+1).fill(null).map((_,i) => i)
	$: allFrequencies = Array(maxFilterFrequency+1).fill(null).map((_,i) => i)
	
	let dragging = false
	
	function evtSetFilter(evt) {
		if(evt.currentTarget.hasAttribute('data-freq') && evt.currentTarget.hasAttribute('data-step')) {
			filterFrequency = 1*evt.currentTarget.getAttribute('data-freq')
			filterStep = 1*evt.currentTarget.getAttribute('data-step')
			dragging = true
		}
	}
	
	function evtDragFilter(evt) {
		if(dragging && evt.currentTarget.hasAttribute('data-freq') && evt.currentTarget.hasAttribute('data-step')) {
			filterFrequency = 1*evt.currentTarget.getAttribute('data-freq')
			filterStep = 1*evt.currentTarget.getAttribute('data-step')
		}
	}
	
	function evtDragEnd() {
		dragging = false
	}
</script>

<style>
	.cpair{
		display: flex;
		flex-direction: column;
		gap: 2px;
		align-items: center;
		flex-shrink: 0;
	}
	
	.cpair::before, .cpair::after {
		content: '';
		width: 9px;
		height: 9px;
		background: grey;
	}
	
	.cpair::after {
		background-color: hsl(calc(255* var(--c-phase)), 100%, 50%);
		border-radius: 8px;
		width: 4px;
		height: 4px;
	}
	.cpair::before {
		background-color: hsl(0, 0%, calc(100% * var(--c-mag)));
		border-radius: 1px;
	}
	
	.cgrid {
		display: grid;
		grid-template-columns: repeat(var(--sample-count, auto-fill), 10px);
		padding: 15px;
		gap: 10px 1px;
		background: black;
		justify-content: start;
		justify-self: stretch;
	}
	
	.cgrid > [data-n="0"] {
		grid-column: 1;
	}
	.clabel {
		grid-column: span 1 / 1;
		color: #fff;
		font-size: 12px;
		text-align: end;
		align-self: start;
		line-height: 10px;
		padding-right: 1em;
	}
	
	.cshift {
		grid-column: span calc(var(--c-shift));
		border: 1px solid cyan;
		background-color: #0ff7;
		border-top: 1px solid transparent;
		display: flex;
		justify-content: end;
		overflow: visible;
		align-self: start;
		padding: 2px 0;
		height: 6px;
	}
	
	
	.fade {
		opacity: 0.3;
		pointer-events: none;
	}
	
	button {
		margin: 0;
	  	padding: 0;
		color: #fff;
		background: #bbb;
		border: none;
		width: 10px;
		height: 10px;
		font-size: 10px;
		display: block;
		cursor: pointer;
		border-radius: 1px;
	}
	
	label {
		display: inline;
	}
	
	table {
		font-size: 12px;
	}
	
	th {
		font-weight: normal;
	}
	
	button.active {
		background: orange;
	}
	
	button:disabled {
		opacity: 0.5;
		cursor: default;
		pointer-events: none;
	}
	
	.rottext {
		writing-mode: vertical-rl;
		text-orientation: sideways;
		width: 1.5em;
	}
	
	fieldset {
		background: black;
		color: #fff;
		border: none;
		margin: 0 0 1em;
		font-size: 12px;
		padding: 0;
	}
	
	legend {
		position: relative;
		background: black;
		display: block;
		padding: 2px 4px;
		margin-bottom: 1em;
		width: 100%;
		box-sizing: border-box;
	}
	
	dl {
		display: grid;
		grid-template-columns: 15em auto;
		align-items: center;
		gap: 0.5em;
		margin: 0;
	}
	
	dt {
		justify-self: end;
		margin: 0;
		text-align: right;
	}
	dd {
		margin: 0;
	}
	
	.container {
		display: grid;
		grid-auto-flow: column;
		grid-auto-columns: 1fr;
		gap: 1em;
		padding: 1em 2em;
		margin: auto;
		max-width: 60em;
		justify-content: stretch;
	}
	
	.cgraph {
		grid-column: 1 / -1;
		border-bottom: 1px solid white;
	}
	
	.cgraph-filter {
		grid-column: var(--c-shift, 1) / span var(--signal-length);
		border-bottom: 1px solid white;
	}
	
	:global(body) {
		background: #444;
		color: #fff;
	}
	
	table {
		background: black;
		color: #fff;
	}

	h2 {
		margin: 0 0 0.5em 0;
		font-weight: normal;
		font-size: 22px;
	}
	
	h1 {
		text-align: center;
		font-weight: normal;
	}

	.marker {
		border: 1px dashed #ffaa00;
		border-top: none;
		border-bottom: none;
		grid-row: 1 / span 4;
		grid-column: var(--c-shift, 1) / span var(--signal-length);
		opacity: 0.7;
		background: #ffaa0033;
	}
</style>

<svelte:window on:mouseup|capture={evtDragEnd} />

<h1>
	Time Frequency Trade-off
</h1>


<div class="container">
	<div style:align-self="stretch" style:flex-grow="1">
<h2>
	Signal and Filter aligment
</h2>

<div class="cgrid" style:--sample-count={signalLength}>
	<svg style:grid-row="1" class="cgraph" viewBox="0 -25 {10*signalLength} 50">
	<path d={'M' + signal.map(({mag,phase},i) => (i*10+5) +','+-25*Math.sin(phase)*mag).join("L ")} stroke="grey" fill="none" vector-effect="non-scaling-stroke" />
	<path d={'M' + signal.map(({mag,phase},i) => (i*10+5) +','+-25*Math.cos(phase)*mag).join("L ")} stroke="white" fill="none" vector-effect="non-scaling-stroke" />
</svg>
	<div style:grid-row="2" class="clabel">
		Signal
	</div>
	{#each signal as {mag,phase}, n}
		<div  style:grid-column="{n+1}" style:grid-row="2" data-n={n} class="cpair" style:--c-mag="{mag}" style:--c-phase="{phase/2/Math.PI}"></div>
	{/each}
	
	<div style:grid-row="3" class="clabel">
		Shift Length
	</div>
	<div style:grid-row="3"  style:grid-column="1" class="cshift" style:--c-shift="{windowShiftSize}"></div>

	<div style:grid-row="4" class="clabel"></div>
		
	<svg style:grid-column-start="{windowShiftSize*filterStep + 1}" style:grid-row="4" style:--c-shift="{windowShiftSize*filterStep+1}" style:--signal-length={windowLength} class="cgraph-filter" viewBox="0 -25 {10*windowLength} 50">
	<path d={'M' + filterWindow.map(({mag,phase},i) => (i*10+5) +','+-25*Math.sin(phase)*mag).join("L ")} stroke="grey" fill="none" vector-effect="non-scaling-stroke" />
	<path d={'M' + filterWindow.map(({mag,phase},i) => (i*10+5) +','+-25*Math.cos(phase)*mag).join("L ")} stroke="white" fill="none" vector-effect="non-scaling-stroke" />
</svg>
	<div class="clabel" style:grid-row="5">
		Filter
	</div>
	{#each Array(filterStep).fill(null) as s}
	  	<div style:grid-row="5" class="cshift" style:--c-shift="{windowShiftSize}"></div>
	{/each}
	{#each filterWindow as {mag,phase}, n}
		<div style:grid-row="5" class="cpair" style:--c-mag="{mag}" style:--c-phase="{phase/2/Math.PI}"></div>
	{/each}
	<div class="marker" style:--c-shift="{windowShiftSize*filterStep+1}" style:--signal-length={windowLength}></div>
</div>
		</div>
</div>

<div class="container">
	
<div>
	<h2>
		Parameters
	</h2>
<fieldset>
	<legend>
		Options
	</legend>
	
	<dl>
		<dt>Signal Length 2^{signalLengthExp}</dt>
		<dd><input type="range" min="0" max="6" bind:value={signalLengthExp}></dd>
		<dt>Filter Length 2^{windowLengthExp} {#if oddLength}+1{/if}<br>
		<label><input type="checkbox" bind:checked={oddLength}/> odd</label></dt>
		<dd><input type="range" min="0" max="6" bind:value={windowLengthExp}></dd>
		<dt>Filter shift length: {windowShiftSize} / {Math.pow(2, signalLengthExp)}<br>
			 <label><input type="checkbox" bind:checked={autoShiftSize}/> auto</label>
			 <span class:fade={!autoShiftSize}> <label><input disabled={!autoShiftSize} type="checkbox" bind:checked={maxShiftSize}/> max</label></span></dt>
		<dd style:accent-color="cyan"><input disabled={autoShiftSize} type="range" min="1" max="{Math.pow(2, signalLengthExp)}" bind:value={windowShiftSize}></dd>
		<dt>Filter shape</dt>
		<dd>
			<label><input type="radio" bind:group={filterShape} value="rect"> Rectangle</label>
			<label><input type="radio" bind:group={filterShape} value="gauss"> Gaussian</label>
		</dd>
	</dl>
</fieldset>


<fieldset>
	<legend>
		Steps
	</legend>
	
	<dl style:accent-color="orange">
		<dt>Filter Step: {filterStep}/{filterStepMax}</dt>
		<dd><input type="range" min="0" max="{filterStepMax}" bind:value={filterStep}></dd>
		
		<dt>Filter Frequency: {filterFrequency}/{maxFilterFrequency}<br><label><input type="checkbox" bind:checked={autoFrequency}/> auto</label></dt>
		<dd><input disabled={autoFrequency} type="range" min="0" max="{maxFilterFrequency}" bind:value={filterFrequency}></dd>
	</dl>
</fieldset>

<div>
	<h2>Explanation</h2>
	<p>
		Above you can a signal a filter. Both are complex values and represented along the time in both cartesian (real and imaginary part are white curve and gray curve) and polar coordinates (square brightness represents the magnitude, color represents the phase)
	</p>
	<p>
		You can configure the size and shape of the filter and shift it across the signal. Depending on size of the filter and the shift length the filter can be placed at a number of positions along the signal. At each possible position the dot product between the filter values and the signal values at that position can be calculated as a measure of how well the filter fits the signal or rather how similar the filter shape is to the the signal.
	</p>
	<p>
		Also depending on length of the filter the filter can represent a number of frequencies. A long filter consists of a higher number of values that can be used to form a higher number of sinusoids.
	</p>
	<p>
		All possible combinations filter positions and filter frequencies are shown in the time frequency grid. As you can see the length of the filter influences the number of positions and the number of frequencies. The total number of bins in the grid is limited.
	</p>
	<p>
		A short filter can only detect high frequencies because low frequencies (long period lengths) do not fit inside the filter. A wide filter can detect both high and low frequencies because it can represent both high and low frequencies. But the computation cost is increased because more numbers need to be multiplied or it has to be shifted across the signal in larger steps which decreases the temporal resolution.
	</p>
	<p>
		So applying a filter is a trade-off between number of frequencies to match, number of time positions and computational complexity. The classic DFT uses a single filter size that matches length of the signal and a list of a all frequencies that fit into the fitler. The short time fourier transform (STFT) uses a shorter filter of fixed size and fixed shift length to slide across the signal to then compute all resulting time/frequency bins. So the STFT decreases the number of frequncy bins in order to increase the number of time bins compared to the DFT. The wavelet transform slides filters of different lenths across the signal but uses only a single fixed filter frequency per filter size. Typically for each filter size only the lowest (none zero) frequency is used because higher frequencies are detected by the shorter filters anyway.
	</p>
	<p>
		In that way the wavelet transform targets the optimal trade-off between time and frequency resolutions. Check the <em>audo</em> option for the filter frequency and the <em>auto</em> and <em>max</em>  options for the shift length and then play with the <em>filter length</em> option to see which frequency bins the wavelet transform calculates.
	</p>
	<p>
		One additional perspective to consider is the following: The original signal consists only of N=<strong>{signalLength}</strong> samples. So whatever the filter does it can not extract more information from the signal than it already contains. So any more than <span>{signalLength}</span> result bins hint at a waste of computation and redundancy. When maximizing the shift length  the time/frequency grid consists of 64 bins for any chosen window size. When - as in the wavelet transform - you take only a single filter frequency per filter size you can see how the selected bins in the grid do not overlap and they approach to cover the whole time frequency plain.
	</p>
</div>
</div>

<div>


<h2>
	Time/Frequency Grid
</h2>

<table cellspacing="0" cellpadding="0">
	<tr>
	<th width="10">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</th>
	<th align="left">0</th>
	<th>Time</th>
	<th align="right">{signalLength}</th>
	<th width="10">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</th>

	</tr>
	<tr>
		<td valign="top" align="center">0</td>
		<td colspan="3" rowspan="3" valign="center" align="center">
			<table cellspacing="1" cellpadding="0">
	{#each allFrequencies as f, fi}
	<tr>
		{#each allSteps as t}
		<td><button style:width="{(7*((signalLength-omittedCount)/allSteps.length)-1)}px" style:height="{Math.floor(7*(signalLength/allFrequencies.length)-1)}px" on:mouseup|capture={evtDragEnd} disabled={f!=filterFrequency&&autoFrequency} class:active={f==filterFrequency && t==filterStep} data-freq={f} data-step={t} on:mousedown={evtSetFilter} on:mouseenter={evtDragFilter}></button></td>
		{/each}
		
		{#if omittedCount}
		<td><button disabled="disabled" style:width="{Math.floor(7*omittedCount-1)}px" style:height="{Math.floor(7*(signalLength/allFrequencies.length)-1)}px"></button></td>
		{/if}
	</tr>
	{/each}
</table>	
		</td>
		<td valign="top" align="center">0</td>
	</tr>
	<tr>
		<th class="rottext" width="10">Freq.</th>
		<th class="rottext" width="10">Freq.</th>
	</tr>
	<tr>
		<td valign="bottom" align="center">{signalLength}</td>
		<td valign="bottom" align="center">{signalLength}</td>
	</tr>
	<tr>
	<th width="10">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</th>
	<th align="left" valign="top">0</th>
	<th align="center" valign="top">Time</th>
	<th align="right" valign="top">{signalLength}</th>
	</tr>
</table>	
</div>
</div>
