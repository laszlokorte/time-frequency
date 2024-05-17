<script>
	function cMake(mag = 0, phase = 0) {
		return { mag, phase };
	}

	function cMul(a, b) {
		return { mag: a.mag * b.mag, phase: a.phase + b.phase };
	}

	function cSum(a, b) {
		const are = Math.cos(a.phase) * a.mag;
		const bre = Math.cos(b.phase) * b.mag;
		const aim = Math.sin(a.phase) * a.mag;
		const bim = Math.sin(b.phase) * b.mag;
		const sumre = are + bre;
		const sumim = aim + bim;
		return {
			mag: Math.sqrt(sumre * sumre + sumim * sumim),
			phase: Math.atan2(aim + bim, are + bre),
		};
	}

	function cMag(z) {
		return z.mag;
	}

	function cPhase(z) {
		return z.phase;
	}

	var seed = 1;
	function random() {
		var x = Math.sin(seed++) * 10000;
		return x - Math.floor(x);
	}

	let gridStyle = "mag";

	let signalLengthExp = 6;
	$: signalLength = Math.pow(2, signalLengthExp);
	$: signal = Array(signalLength)
		.fill(null)
		.map((_, i) =>
			cMake(random(), Math.sin((i / 37) * 2 * Math.PI) * 2 * Math.PI),
		);
	$: maxMag = signal.reduce(
		(acc, { mag }) => Math.max(Math.abs(mag), acc),
		0,
	);

	let filterShape = "rect";

	let wantOddLength = false;
	let windowLengthExp = 3;
	$: canOddLength =
		true || !(windowLengthExp == 0 || windowLengthExp == signalLengthExp);
	$: oddLength =
		(wantOddLength && canOddLength) ^ (windowLengthExp == 0) &&
		windowLengthExp != signalLengthExp;
	$: windowLengthExp = Math.max(
		autoFrequency ? 1 : 0,
		Math.min(windowLengthExp, signalLengthExp),
	);
	$: windowLength = Math.min(
		signalLength,
		Math.pow(2, windowLengthExp) + (oddLength ? 1 : 0),
	);

	let autoShiftSize = false;
	let maxShiftSize = false;
	$: windowShiftSize = 1;
	$: windowShiftSize = autoShiftSize
		? maxShiftSize
			? windowLength
			: Math.ceil(windowLength / 2)
		: Math.min(windowShiftSize, windowLength);

	let filterStep = 0;
	$: filterStepMax = Math.floor(
		(signalLength - windowLength) / windowShiftSize,
	);
	$: filterStep = Math.min(filterStep, filterStepMax);

	$: omittedCount =
		signalLength - (filterStepMax * windowShiftSize + windowLength);

	let autoFrequency = false;
	let filterFrequency = 0;
	$: maxFilterFrequency =
		Math.pow(2, windowLengthExp) - 1 + (oddLength ? 1 : 0);
	$: filterFrequency = autoFrequency
		? 1
		: Math.min(filterFrequency, maxFilterFrequency);
	$: filterWindow = Array(windowLength)
		.fill(null)
		.map((_, i) =>
			cMake(
				filterShape == "rect"
					? 1
					: Math.exp(
							-Math.pow(
								((i + 0.5) / (windowLength + 1) - 0.5) * 4,
								2,
							),
						),
				(((filterFrequency > maxFilterFrequency / 2
					? filterFrequency - maxFilterFrequency - 1
					: filterFrequency) *
					i) /
					windowLength) *
					2 *
					Math.PI,
			),
		);

	$: allSteps = Array(filterStepMax + 1)
		.fill(null)
		.map((_, i) => i);
	$: allFrequencies = Array(
		maxFilterFrequency +
			(maxFilterFrequency ? 1 : 0) +
			(!oddLength ? 1 : 0),
	)
		.fill(null)
		.map((_, i, all) =>
			i - (!oddLength ? 1 : 0) <= maxFilterFrequency / 2
				? (all.length % 2) -
					1 -
					i +
					(maxFilterFrequency + (maxFilterFrequency & 1)) / 2
				: (all.length % 2) +
					maxFilterFrequency -
					i +
					(maxFilterFrequency + (maxFilterFrequency & 1)) / 2,
		);

	$: dotProducts = allSteps.map((s) =>
		allFrequencies.map((f) => {
			return filterWindow
				.map((_, o) =>
					cMul(
						cMake(
							filterShape == "rect"
								? 2 / Math.sqrt(2 * filterWindow.length)
								: (Math.exp(
										-Math.pow(
											((o + 0.5) / (windowLength + 1) -
												0.5) *
												4,
											2,
										),
									) *
										2) /
										Math.sqrt(windowLength),
							(((f > maxFilterFrequency / 2
								? f - maxFilterFrequency - 1
								: f) *
								-o) /
								windowLength) *
								2 *
								Math.PI,
						),
						signal[s * windowShiftSize + o],
					),
				)
				.reduce(cSum, cMake());
		}),
	);

	let dragging = false;

	function evtSetFilter(evt) {
		if (
			evt.currentTarget.hasAttribute("data-freq") &&
			evt.currentTarget.hasAttribute("data-step")
		) {
			filterFrequency = 1 * evt.currentTarget.getAttribute("data-freq");
			filterStep = 1 * evt.currentTarget.getAttribute("data-step");
			dragging = true;
		}
	}

	function evtDragFilter(evt) {
		if (
			dragging &&
			evt.currentTarget.hasAttribute("data-freq") &&
			evt.currentTarget.hasAttribute("data-step")
		) {
			filterFrequency = 1 * evt.currentTarget.getAttribute("data-freq");
			filterStep = 1 * evt.currentTarget.getAttribute("data-step");
		}
	}

	function evtDragEnd() {
		dragging = false;
	}
</script>

<h1>Time Frequency Trade-off</h1>

<div class="container">
	<div style:align-self="stretch" style:flex-grow="1">
		<h2>Signal and Filter aligment</h2>

		<div class="cgrid" style:--sample-count={signalLength}>
			<svg
				style:grid-row="1"
				class="cgraph"
				viewBox="0 -25 {10 * signalLength} 50"
			>
				<path
					d={"M" +
						signal
							.map(
								({ mag, phase }, i) =>
									i * 10 +
									5 +
									"," +
									-25 * Math.sin(phase) * mag,
							)
							.join("L ")}
					stroke="grey"
					fill="none"
					vector-effect="non-scaling-stroke"
				/>
				<path
					d={"M" +
						signal
							.map(
								({ mag, phase }, i) =>
									i * 10 +
									5 +
									"," +
									-25 * Math.cos(phase) * mag,
							)
							.join("L ")}
					stroke="white"
					fill="none"
					vector-effect="non-scaling-stroke"
				/>
			</svg>
			<div style:grid-row="2" class="clabel">Signal</div>
			{#each signal as { mag, phase }, n}
				<div
					style:grid-column={n + 1}
					style:grid-row="2"
					data-n={n}
					class="cpair"
					style:--c-mag={mag}
					style:--c-phase={phase / 2 / Math.PI}
				></div>
			{/each}

			<div style:grid-row="3" class="clabel">Shift Length</div>
			<div
				style:grid-row="3"
				style:grid-column-start="1"
				class="cshift"
				style:--c-shift={windowShiftSize}
			></div>

			<div style:grid-row="4" class="clabel"></div>

			<svg
				style:grid-column-start={windowShiftSize * filterStep + 1}
				style:grid-row="4"
				style:--c-shift={windowShiftSize * filterStep + 1}
				style:--signal-length={windowLength}
				class="cgraph-filter"
				viewBox="0 -25 {10 * windowLength} 50"
			>
				<path
					d={"M" +
						filterWindow
							.map(
								({ mag, phase }, i) =>
									i * 10 +
									5 +
									"," +
									-25 * Math.sin(phase) * mag,
							)
							.join("L ")}
					stroke="grey"
					fill="none"
					vector-effect="non-scaling-stroke"
				/>
				<path
					d={"M" +
						filterWindow
							.map(
								({ mag, phase }, i) =>
									i * 10 +
									5 +
									"," +
									-25 * Math.cos(phase) * mag,
							)
							.join("L ")}
					stroke="white"
					fill="none"
					vector-effect="non-scaling-stroke"
				/>
			</svg>
			<div class="clabel" style:grid-row="5">Filter</div>
			{#each Array(filterStep).fill(null) as s}
				<div
					style:grid-row="5"
					class="cshift"
					style:--c-shift={windowShiftSize}
				></div>
			{/each}
			{#each filterWindow as { mag, phase }, n}
				<div
					style:grid-row="5"
					class="cpair"
					style:--c-mag={mag}
					style:--c-phase={phase / 2 / Math.PI}
				></div>
			{/each}
			<div
				class="marker"
				style:--c-shift={windowShiftSize * filterStep + 1}
				style:--signal-length={windowLength}
			></div>
		</div>
	</div>
</div>

<div class="container">
	<div>
		<h2>Parameters</h2>
		<fieldset>
			<legend> Options </legend>

			<dl>
				<dt>Signal Length 2^{signalLengthExp}</dt>
				<dd>
					<input
						type="range"
						min="0"
						max="6"
						bind:value={signalLengthExp}
					/>
				</dd>
				<dt>
					Filter Length 2^{windowLengthExp}
					{#if oddLength}+1{/if}<br />
					<label
						><input
							class:hidden={!canOddLength}
							disabled={!canOddLength}
							type="checkbox"
							bind:checked={wantOddLength}
						/> odd</label
					>
				</dt>
				<dd>
					<input
						type="range"
						min="0"
						max="6"
						bind:value={windowLengthExp}
					/>
				</dd>
				<dt>
					Filter shift length: {windowShiftSize} / {Math.pow(
						2,
						signalLengthExp,
					)}<br />
					<label
						><input type="checkbox" bind:checked={autoShiftSize} /> auto</label
					>
					<span class:fade={!autoShiftSize}>
						<label
							><input
								disabled={!autoShiftSize}
								type="checkbox"
								bind:checked={maxShiftSize}
							/> max</label
						></span
					>
				</dt>
				<dd style:accent-color="cyan">
					<input
						disabled={autoShiftSize}
						type="range"
						min="1"
						max={Math.pow(2, signalLengthExp)}
						bind:value={windowShiftSize}
					/>
				</dd>
				<dt>Filter shape</dt>
				<dd>
					<label class="cb-label"
						><input
							type="radio"
							bind:group={filterShape}
							value="rect"
						/> Rectangle</label
					>
					<label class="cb-label"
						><input
							type="radio"
							bind:group={filterShape}
							value="gauss"
						/> Gaussian</label
					>
				</dd>
			</dl>
		</fieldset>

		<fieldset>
			<legend> Steps </legend>

			<dl style:accent-color="orange">
				<dt>Filter Step: {filterStep}/{filterStepMax}</dt>
				<dd>
					<input
						type="range"
						min="0"
						max={filterStepMax}
						bind:value={filterStep}
					/>
				</dd>

				<dt>
					Filter Frequency: {filterFrequency}/{maxFilterFrequency}<br
					/><label
						><input type="checkbox" bind:checked={autoFrequency} /> auto</label
					>
				</dt>
				<dd>
					<input
						disabled={autoFrequency}
						type="range"
						min="0"
						max={maxFilterFrequency}
						bind:value={filterFrequency}
					/>
				</dd>
			</dl>
		</fieldset>
	</div>

	<div>
		<h2>Time/Frequency Grid</h2>

		<table cellspacing="0" cellpadding="0" width="100%">
			<tr>
				<th width="10">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</th>
				<th align="left">0</th>
				<th>Time</th>
				<th align="right">{signalLength}</th>
				<th width="10">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</th>
			</tr>
			<tr>
				<td valign="top" align="center">{signalLength / 2}</td>
				<td colspan="3" rowspan="3" valign="center" align="center">
					<div
						class="tf-grid"
						class:even={!oddLength}
						style:--odd-row-count={allFrequencies.length -
							(oddLength ? 0 : 2)}
					>
						{#each allFrequencies as f, fi}
							{#each allSteps as t, ti}
								<button
									data-t={t}
									style:grid-column={t + 1}
									on:pointerup|capture={evtDragEnd}
									disabled={f != filterFrequency &&
										(-f + windowLength) % windowLength !=
											filterFrequency &&
										autoFrequency}
									class:active={f == filterFrequency &&
										t == filterStep}
									class:active-secondary={(-f +
										windowLength) %
										windowLength ==
										filterFrequency && t == filterStep}
									data-freq={f}
									data-step={t}
									on:pointerdown={evtSetFilter}
									on:pointerenter={evtDragFilter}
									style:--cmag={cMag(dotProducts[ti][fi])}
									style:--cphase={cPhase(
										dotProducts[ti][fi],
									) /
										Math.PI /
										2}
									class:showmag={gridStyle == "mag"}
									class:showphase={gridStyle == "phase"}
								></button>
							{/each}

							{#if omittedCount}
								<button
									style:grid-column={allSteps.length + 1}
									disabled="disabled"
								></button>
							{/if}
						{/each}
					</div>
				</td>
				<td valign="top" align="center">{signalLength / 2}</td>
			</tr>
			<tr>
				<th class="rottext" width="10">Freq.</th>
				<th class="rottext" width="10">Freq.</th>
			</tr>
			<tr>
				<td valign="bottom" align="center"
					>{-signalLength / 2 + (oddLength ? 0 : 1)}</td
				>
				<td valign="bottom" align="center"
					>{-signalLength / 2 + (oddLength ? 0 : 1)}</td
				>
			</tr>
			<tr>
				<th width="10">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</th>
				<th align="left" valign="top">0</th>
				<th align="center" valign="top">Time</th>
				<th align="right" valign="top">{signalLength}</th>
			</tr>
		</table>

		<div
			style:justify-content="center"
			style:padding="0.5em"
			style:display="flex"
			style:gap="1em"
		>
			<label
				><input type="radio" value="none" bind:group={gridStyle} /> Empty</label
			>
			<label
				><input type="radio" value="mag" bind:group={gridStyle} /> Magnitude</label
			>
			<label
				><input type="radio" value="phase" bind:group={gridStyle} /> Phase</label
			>
		</div>
	</div>
</div>
<div class="container text-page">
	<div>
		<h2>Explanation</h2>
		<p>
			Above you can a signal a filter. Both are complex values and
			represented along the time in both cartesian (real and imaginary
			part are white curve and gray curve) and polar coordinates (square
			brightness represents the magnitude, color represents the phase)
		</p>

		<p>
			You can configure the size and shape of the filter and shift it
			across the signal. Depending on size of the filter and the shift
			length the filter can be placed at a number of positions along the
			signal. At each possible position the dot product between the filter
			values and the signal values at that position can be calculated as a
			measure of how well the filter fits the signal or rather how similar
			the filter shape is to the the signal.
		</p>
		<p>
			Also depending on length of the filter the filter can represent a
			number of frequencies. A long filter consists of a higher number of
			values that can be used to form a higher number of sinusoids.
		</p>
		<p>
			All possible combinations filter positions and filter frequencies
			are shown in the time frequency grid. As you can see the length of
			the filter influences the number of positions and the number of
			frequencies. The total number of bins in the grid is limited.
		</p>
		<p>
			A short filter can only detect high frequencies because low
			frequencies (long period lengths) do not fit inside the filter. A
			wide filter can detect both high and low frequencies because it can
			represent both high and low frequencies. But the computation cost is
			increased because more numbers need to be multiplied or it has to be
			shifted across the signal in larger steps which decreases the
			temporal resolution.
		</p>
		<p>
			So applying a filter is a trade-off between number of frequencies to
			match, number of time positions and computational complexity. The
			classic DFT uses a single filter size that matches length of the
			signal and a list of a all frequencies that fit into the fitler. The
			short time fourier transform (STFT) uses a shorter filter of fixed
			size and fixed shift length to slide across the signal to then
			compute all resulting time/frequency bins. So the STFT decreases the
			number of frequncy bins in order to increase the number of time bins
			compared to the DFT. The wavelet transform slides filters of
			different lenths across the signal but uses only a single fixed
			filter frequency per filter size. Typically for each filter size
			only the lowest (none zero) frequency is used because higher
			frequencies are detected by the shorter filters anyway.
		</p>
		<p>
			In that way the wavelet transform targets the optimal trade-off
			between time and frequency resolutions. Check the <em>audo</em>
			option for the filter frequency and the <em>auto</em> and
			<em>max</em>
			options for the shift length and then play with the
			<em>filter length</em> option to see which frequency bins the wavelet
			transform calculates.
		</p>
		<p>
			One additional perspective to consider is the following: The
			original signal consists only of N=<strong>{signalLength}</strong>
			samples. So whatever the filter does it can not extract more information
			from the signal than it already contains. So any more than
			<span>{signalLength}</span> result bins hint at a waste of computation
			and redundancy. When maximizing the shift length the time/frequency grid
			consists of 64 bins for any chosen window size. When - as in the wavelet
			transform - you take only a single filter frequency per filter size you
			can see how the selected bins in the grid do not overlap and they approach
			to cover the whole time frequency plain.
		</p>
	</div>
</div>

<style>
	.cpair {
		display: flex;
		flex-direction: column;
		gap: 2px;
		align-items: center;
		flex-shrink: 0;
	}

	.cpair::before,
	.cpair::after {
		content: "";
		width: 9px;
		height: 9px;
		background: grey;
	}

	.cpair::after {
		background-color: hsl(calc(360 * var(--c-phase)), 100%, 50%);
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
		grid-template-rows: 3fr 1fr 1fr 3fr 1fr;
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
		grid-column-end: span calc(var(--c-shift));
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
		font-size: 10px;
		display: block;
		cursor: pointer;
		border-radius: 1px;
	}

	.showmag {
		background-color: hsl(0, 50%, calc(var(--cmag) * 50%));
	}
	.showphase {
		background-color: hsl(calc(360 * var(--cphase)), 60%, 50%);
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

	button.active-secondary {
		outline: 4px dotted orange;
		z-index: 10;
	}
	button.active {
		outline: 4px solid orange;
		z-index: 10;
	}

	button:disabled {
		opacity: 0.5;
		cursor: default;
		pointer-events: none;
		background: #bbb;
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
		padding: 0 0 1em;
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
		grid-template-columns: 1fr;
		grid-auto-columns: 1fr;
		gap: 1em;
		padding: 0 2em;
		margin: 2em auto;
		max-width: 62em;
		justify-content: stretch;
	}

	.cgraph {
		grid-column: 1 / -1;
		border-bottom: 1px solid white;
		height: 60px;
		width: 100%;
	}

	.cgraph-filter {
		grid-column: var(--c-shift, 1) / span var(--signal-length);
		border-bottom: 1px solid white;
		height: 60px;
		width: 100%;
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

	.cb-label {
		padding: 0.3em;
	}

	.tf-grid {
		display: grid;
		grid-auto-columns: 1fr;
		grid-auto-flow: row;
		aspect-ratio: 1;
		align-items: stretch;
		justify-items: stretch;
		gap: 1px;
	}

	.tf-grid {
		grid-template-rows: repeat(var(--odd-row-count), 1fr);
	}

	.tf-grid.even {
		grid-template-rows: 0.5fr repeat(var(--odd-row-count), 1fr) 0.5fr;
		aspect-ratio: none;
	}

	.tf-grid > [data-t="0"] {
		grid-column: 1;
	}

	.text-page {
		background: #fafafa;
		color: #000;
		padding: 1.5em 2em;
		box-sizing: border-box;
	}
</style>
