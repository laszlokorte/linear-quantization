<script>
	const formatter = new Intl.NumberFormat('en-IN', { maximumFractionDigits: 3, minimumFractionDigits : 2, useGrouping: false })
	
	const signalGenerators = {
		linear: (v, period, amplitude) => Math.sign(amplitude) * (((v/2*period) % amplitude + amplitude)%amplitude * 2 - amplitude),
		sine: (v, period, amplitude) => amplitude * Math.sin(v*period*Math.PI),
	}

	let sr = 500
	let stepBits = 5
	$: steps = Math.pow(2, stepBits)
	let bias = 0
	let period = 1
	let amplitude = 1

	let signalType = 'linear'
	
	$: binSize = 2 / (steps-1)
	$: bins = Array(steps).fill(0).map((_,b) => -1 + (b+0.5)*binSize - binSize*(bias/2+0.5))
	
	
	$: domain = Array(sr).fill(0).map((_,i,all) => 2*(i/all.length)-1)
	$: signal = domain.map((v) => signalGenerators[signalType](v, period, amplitude))
	$: quan = signal.map(v => bins.reduce((acc, b) => Math.abs(b-v)<=Math.abs(acc-v)?b:acc, Infinity))
	$: err = signal.map((v,i) => (quan[i]-signal[i]))
	$: avgErr = err.slice(1).reduce((a,b) => a+b, 0) / err.length
	
	$: signalPower = signal.reduce((acc, v) => acc + v*v, 0) / signal.length
	$: errorPower = (err.reduce((acc, v) => acc + v*v, 0) || 1) / err.length
</script>

<style>
	dl {
		display: grid;
		grid-template-columns: [first-start]auto[first-end] auto auto;
		justify-content: start;
		margin: 0;
		padding: 0;
	}
	
	dt,dl {
		margin: 0;
		padding: 0;
	}
	
	dt {
		grid-column: first;
	}

	footer {
		margin: 3em;
		text-align: center;
	}

</style>

<h1>
	Linear Quantization Error
</h1>

<p>
	Below you can explore how quantizing a signal introduces an error. The difference between the original signal and the quantized signal can be seen as additive noise.
</p>

<p>
	By playing around with the signal and quantization parameters you can see that the power of the resulting noise does not depend on the signal frequency but only on the quantization method (number of bits) and if the signal 
</p>

<fieldset>
	<legend>
		Parameters
	</legend>
	
	
<dl>
	<dt>Signal Type</dt>
	<dd style="display: flex; gap: 2em">
		{#each Object.keys(signalGenerators) as type}
		<label style="padding: 0.5em 0"><input type="radio" value={type}  bind:group={signalType} /> {type}</label>
		{/each}
	</dd>

	<dt><label for="period">Signal Frequency</label></dt>
	<dd><input id="period" type="range" min="1" max="10" bind:value={period} /></dd>
	<dd><output>{period}</output></dd>
	
	<dt><label for="period">Signal Amplitude</label></dt>
	<dd><input id="period" type="range" min="-1.5" max="1.5" step="0.01" bind:value={amplitude} /></dd>
	<dd><output on:dblclick={() => {amplitude = 1}}>{amplitude}</output></dd>
	
	<dt><label for="steps">Quantization Bits</label></dt>
	<dd><input id="steps" type="range" min="1" max="7" bind:value={stepBits} /></dd>
	<dd><output>{stepBits} bits ({steps} bins)</output></dd>
	
	<dt><label for="bias">Quantization Bias</label></dt>
	<dd><input id="bias" type="range" min="-5" max="5" step="0.01" bind:value={bias} /></dd>
	<dd><output on:dblclick={() => {bias = 0}}>{bias}</output></dd>
	
	<dt style="color: green">Signal Power</dt>
	<dd style="color: green"><output>{formatter.format(signalPower)}</output></dd>
	
	<dt style="color: red">Error Power</dt>
	<dd style="color: red"><output>{formatter.format(errorPower)}</output></dd>
	
	<dt>Signal/Noise-Ratio (SNR)</dt>
	<dd><output>{formatter.format(signalPower/errorPower)}</output></dd>
	
	<dt>SNR (dB)</dt>
	<dd><output>{formatter.format(10 * Math.log10(signalPower/errorPower))}</output></dd>
</dl>
</fieldset>





<svg viewBox="-220 -150 440 300">
	<g>
		<line x1="-210" y1="0" x2="210" y2="0" stroke="black" stroke-width="0.5" />
		<line y1="-120" x1="0" y2="140" x2="0" stroke="black" stroke-width="0.5" />
		<polygon points="210 0 204 4 204 -4" fill="black" />
		<polygon points="0 -120 -4 -114 4 -114" fill="black" />
		
		<line x1="-210" y1="{-bins[0]*100}" x2="210" y2="{-bins[0]*100}" stroke="black" stroke-width="0.5" stroke-dasharray="5 7" stroke-opacity="0.5" />
		<line x1="-210" y1="{-bins[bins.length-1]*100}" x2="210" y2="{-bins[bins.length-1]*100}" stroke="black" stroke-opacity="0.5" stroke-width="0.5" stroke-dasharray="5 7" />
		
		<text x="190" y="{-bins[0]*100-3}" font-size="7" text-anchor="end" opacity="0.5">maximum Quantiztation value</text>
		<text x="190" y="{-bins[bins.length-1]*100-3}" font-size="7" text-anchor="end" opacity="0.5">minimum Quantiztation value</text>
	</g>

		

	<polyline points="-200,0, {quan.reduce((acc, v, i) => acc + ` ${(domain[i]*200)},${(-v*100)}`,'')} 200 0" stroke-width="0.5" stroke="orange" fill="orange" fill-opacity="0.1" />
	<polyline points="-200,0 {signal.reduce((acc, v, i) => acc + ` ${(domain[i]*200)},${(-v*100)}`,'')} 200 0" fill="green" stroke-width="1" stroke="green" fill-opacity="0.01" />
	
	
	<polyline points="-200,0, {err.reduce((acc, e, i) => acc + ` ${(domain[i]*200)},${(-e*100)}`,'')} 200 0" stroke-width="0.1" stroke="red" fill="red" fill-opacity="0.3" />
	
	<line x1={200} y1={-Math.sqrt(errorPower)*100} x2={220} y2={-Math.sqrt(errorPower)*100} stroke-width="1.5" stroke="red" />
	<line x1={200} y1={-Math.sqrt(signalPower)*100} x2={220} y2={-Math.sqrt(signalPower)*100} stroke-width="1.5" stroke="green" />
</svg>

<footer>
	<p>
		<a href="https://tools.laszlokorte.de">more eduational tools</a>
	</p>
</footer>