<script>
  import { onMount } from 'svelte';
  import Chart from 'chart.js/auto';

  let params = $state({
    numBarrels: 4,
    inputVolume: 500,
    cycleLength: 2,
    simulationMonths: 48,
    barrels: [
      { capacity: 5000, drainVolume: 300, pricePerLiter: 50 },
      { capacity: 3000, drainVolume: 200, pricePerLiter: 80 },
      { capacity: 2000, drainVolume: 150, pricePerLiter: 120 },
      { capacity: 1500, drainVolume: 100, pricePerLiter: 200 }
    ]
  });

  let data = $state([]);
  let priceAnalysisData = $state([]);
  let ageChart = $state();
  let priceChart = $state();
  let ageCanvasElement = $state();
  let priceCanvasElement = $state();
  let updateTimeout = $state();

  const generateBarrelColors = (numBarrels) => {
    const baseColors = ['#8B4513', '#A0522D', '#CD853F', '#D2691E', '#DEB887', '#F4A460', '#DAA520', '#B8860B'];
    return Array.from({ length: numBarrels }, (_, i) => baseColors[i % baseColors.length]);
  };

  let barrelColors = $derived(generateBarrelColors(params.numBarrels));

  const debouncedUpdate = () => {
    if (updateTimeout) {
      clearTimeout(updateTimeout);
    }
    
    updateTimeout = setTimeout(() => {
      // Only run calculations if we have valid numeric values
      const hasValidParams = 
        typeof params.numBarrels === 'number' && params.numBarrels > 0 &&
        typeof params.inputVolume === 'number' && params.inputVolume > 0 &&
        typeof params.cycleLength === 'number' && params.cycleLength > 0 &&
        typeof params.simulationMonths === 'number' && params.simulationMonths > 0 &&
        params.barrels.every(barrel => 
          typeof barrel.capacity === 'number' && barrel.capacity > 0 &&
          typeof barrel.drainVolume === 'number' && barrel.drainVolume >= 0 &&
          typeof barrel.pricePerLiter === 'number' && barrel.pricePerLiter > 0
        );

      if (hasValidParams) {
        runSimulation();
        runPriceAnalysis();
        updateCharts();
      }
    }, 300);
  };

  const runSimulation = () => {
    const { numBarrels, inputVolume, cycleLength, simulationMonths, barrels } = params;
    
    let barrelContents = Array.from({ length: numBarrels }, () => []);
    let results = [];
    
    const numCycles = Math.floor(simulationMonths / cycleLength);
    
    for (let barrelIndex = 0; barrelIndex < numBarrels; barrelIndex++) {
      const barrel = barrels[barrelIndex];
      const parcelsToAdd = Math.floor(barrel.capacity / inputVolume);
      
      for (let i = 0; i < parcelsToAdd; i++) {
        barrelContents[barrelIndex].push({
          volume: inputVolume,
          entryTime: 0,
          age: 0
        });
      }
    }
    
    for (let cycle = 0; cycle <= numCycles; cycle++) {
      const currentTime = cycle * cycleLength;
      
      barrelContents = barrelContents.map(barrel => 
        barrel.map(parcel => ({
          ...parcel,
          age: currentTime - parcel.entryTime
        }))
      );
      
      if (cycle > 0) {
        for (let barrelIndex = 0; barrelIndex < numBarrels; barrelIndex++) {
          if (barrelContents[barrelIndex].length === 0) continue;
          
          barrelContents[barrelIndex].sort((a, b) => a.entryTime - b.entryTime);
          
          let volumeToDrain = barrels[barrelIndex].drainVolume;
          let drainedParcels = [];
          
          while (volumeToDrain > 0 && barrelContents[barrelIndex].length > 0) {
            const parcel = barrelContents[barrelIndex][0];
            if (parcel.volume <= volumeToDrain) {
              drainedParcels.push(parcel);
              volumeToDrain -= parcel.volume;
              barrelContents[barrelIndex].shift();
            } else {
              drainedParcels.push({
                volume: volumeToDrain,
                entryTime: parcel.entryTime,
                age: parcel.age
              });
              parcel.volume -= volumeToDrain;
              volumeToDrain = 0;
            }
          }
        }
        
        let availableTransfers = [params.inputVolume];
        
        for (let barrelIndex = 0; barrelIndex < numBarrels - 1; barrelIndex++) {
          const available = availableTransfers[barrelIndex] - barrels[barrelIndex].drainVolume;
          availableTransfers.push(Math.max(0, available));
        }
        
        for (let barrelIndex = 0; barrelIndex < numBarrels - 1; barrelIndex++) {
          const transferVolume = availableTransfers[barrelIndex + 1];
          if (transferVolume <= 0 || barrelContents[barrelIndex].length === 0) continue;
          
          barrelContents[barrelIndex].sort((a, b) => a.entryTime - b.entryTime);
          
          let volumeToTransfer = transferVolume;
          let transferredParcels = [];
          
          while (volumeToTransfer > 0 && barrelContents[barrelIndex].length > 0) {
            const parcel = barrelContents[barrelIndex][0];
            if (parcel.volume <= volumeToTransfer) {
              transferredParcels.push({
                ...parcel,
                transferTime: currentTime
              });
              volumeToTransfer -= parcel.volume;
              barrelContents[barrelIndex].shift();
            } else {
              transferredParcels.push({
                volume: volumeToTransfer,
                entryTime: parcel.entryTime,
                age: parcel.age,
                transferTime: currentTime
              });
              parcel.volume -= volumeToTransfer;
              volumeToTransfer = 0;
            }
          }
          
          barrelContents[barrelIndex + 1] = [...barrelContents[barrelIndex + 1], ...transferredParcels];
        }
        
        barrelContents[0].push({
          volume: inputVolume,
          entryTime: currentTime,
          age: 0
        });
      }
      
      barrelContents = barrelContents.map((barrel, index) => {
        const capacity = barrels[index].capacity;
        let totalVolume = barrel.reduce((sum, parcel) => sum + parcel.volume, 0);
        
        while (totalVolume > capacity && barrel.length > 0) {
          const removedParcel = barrel.shift();
          totalVolume -= removedParcel.volume;
        }
        
        return barrel;
      });
      
      const calcAvgAge = (barrel) => {
        if (barrel.length === 0) return 0;
        const totalVolumeAge = barrel.reduce((sum, parcel) => sum + (parcel.volume * parcel.age), 0);
        const totalVolume = barrel.reduce((sum, parcel) => sum + parcel.volume, 0);
        return totalVolume > 0 ? totalVolumeAge / totalVolume : 0;
      };
      
      const barrelData = barrelContents.map((barrel, index) => ({
        avgAge: calcAvgAge(barrel),
        volume: barrel.reduce((sum, parcel) => sum + parcel.volume, 0),
        barrelIndex: index
      }));
      
      let totalRevenuePerCycle = 0;
      const barrelRevenues = barrels.map((barrel, index) => {
        const revenuePerCycle = cycle > 0 ? (barrel.drainVolume / 1000) * barrel.pricePerLiter : 0;
        totalRevenuePerCycle += revenuePerCycle;
        return revenuePerCycle;
      });
      
      const totalCycles = Math.max(0, cycle);
      const cumulativeBarrelRevenues = barrelRevenues.map(revenue => revenue * totalCycles);
      const totalCumulativeRevenue = cumulativeBarrelRevenues.reduce((sum, revenue) => sum + revenue, 0);
      
      const valuePerMonth = currentTime > 0 ? totalCumulativeRevenue / currentTime : 0;
      
      const result = {
        month: currentTime,
        totalRevenuePerCycle,
        totalCumulativeRevenue,
        valuePerMonth
      };
      
      barrelData.forEach((barrel, index) => {
        result[`barrel${index}AvgAge`] = barrel.avgAge;
        result[`barrel${index}Volume`] = barrel.volume;
        result[`barrel${index}Revenue`] = barrelRevenues[index];
        result[`barrel${index}CumulativeRevenue`] = cumulativeBarrelRevenues[index];
      });
      
      results.push(result);
    }
    
    data = results;
  };

  const runPriceAnalysis = () => {
    const { cycleLength, barrels } = params;
    
    const priceResults = [];
    
    barrels.forEach((barrel, barrelIndex) => {
      for (let i = 0; i <= 100; i += 10) {
        const priceMultiplier = 0.5 + (i / 100);
        const newPrice = barrel.pricePerLiter * priceMultiplier;
        
        let totalRevenue = 0;
        barrels.forEach((b, index) => {
          const price = index === barrelIndex ? newPrice : b.pricePerLiter;
          totalRevenue += (b.drainVolume / 1000) * price;
        });
        
        priceResults.push({
          barrelIndex,
          barrelName: `Barrel ${barrelIndex + 1}`,
          priceMultiplier,
          actualPrice: newPrice,
          totalRevenue,
          revenuePerMonth: totalRevenue / cycleLength
        });
      }
    });
    
    priceAnalysisData = priceResults;
  };

  const updateAgeChart = () => {
    if (!ageChart || !data.length) {
      return;
    }

    const labels = data.map(d => d.month);
    const datasets = [];
    
    for (let i = 0; i < params.numBarrels; i++) {
      datasets.push({
        label: `Barrel ${i + 1}`,
        data: data.map(d => d[`barrel${i}AvgAge`]),
        borderColor: barrelColors[i],
        backgroundColor: barrelColors[i] + '20',
        borderWidth: 3,
        fill: false,
        tension: 0.1,
        borderDash: i === 0 ? [5, 5] : []
      });
    }

    ageChart.data = { labels, datasets };
    ageChart.update('none');
  };

  const updatePriceChart = () => {
    if (!priceChart || !priceAnalysisData.length) {
      return;
    }

    const datasets = [];
    
    for (let i = 0; i < params.numBarrels; i++) {
      const barrelData = priceAnalysisData.filter(d => d.barrelIndex === i);
      datasets.push({
        label: `Barrel ${i + 1}`,
        data: barrelData.map(d => ({ x: d.priceMultiplier, y: d.totalRevenue })),
        borderColor: barrelColors[i],
        backgroundColor: barrelColors[i] + '20',
        borderWidth: 3,
        fill: false,
        tension: 0.1
      });
    }

    priceChart.data = { datasets };
    priceChart.update('none');
  };

  const updateCharts = () => {
    updateAgeChart();
    updatePriceChart();
  };

  const handleParamChange = (param, value) => {
    // Allow empty values for better UX
    if (value === '' || value === null || value === undefined) {
      params = {
        ...params,
        [param]: ''
      };
      return;
    }

    const newValue = parseFloat(value);
    
    if (param === 'numBarrels') {
      const newNumBarrels = Math.max(1, Math.min(10, Math.floor(newValue) || 1));
      const newBarrels = [...params.barrels];
      
      while (newBarrels.length < newNumBarrels) {
        const lastBarrel = newBarrels[newBarrels.length - 1];
        newBarrels.push({
          capacity: lastBarrel?.capacity || 1000,
          drainVolume: lastBarrel?.drainVolume || 100,
          pricePerLiter: (lastBarrel?.pricePerLiter || 50) * 1.5
        });
      }
      newBarrels.length = newNumBarrels;
      
      params = {
        ...params,
        numBarrels: newNumBarrels,
        barrels: newBarrels
      };
    } else {
      params = {
        ...params,
        [param]: isNaN(newValue) ? value : newValue
      };
    }
    
    // ONLY CHANGE: Call debouncedUpdate directly instead of via effect
    debouncedUpdate();
  };

  const handleBarrelParamChange = (barrelIndex, param, value) => {
    // Allow empty values for better UX
    if (value === '' || value === null || value === undefined) {
      const newBarrels = [...params.barrels];
      newBarrels[barrelIndex] = {
        ...newBarrels[barrelIndex],
        [param]: ''
      };
      params = {
        ...params,
        barrels: newBarrels
      };
      return;
    }

    const newValue = parseFloat(value);
    const newBarrels = [...params.barrels];
    newBarrels[barrelIndex] = {
      ...newBarrels[barrelIndex],
      [param]: isNaN(newValue) ? value : newValue
    };
    params = {
      ...params,
      barrels: newBarrels
    };
    
    // ONLY CHANGE: Call debouncedUpdate directly instead of via effect
    debouncedUpdate();
  };

  // Derived calculations
  const totalDrain = $derived(params.barrels.reduce((sum, b) => sum + b.drainVolume, 0));
  const flowBalance = $derived(params.inputVolume - totalDrain);
  const systemEfficiency = $derived(((totalDrain / params.inputVolume) * 100).toFixed(0));
  const agePremium = $derived((params.barrels[params.numBarrels - 1]?.pricePerLiter / params.barrels[0]?.pricePerLiter).toFixed(1));
  const latestData = $derived(data[data.length - 1]);

  // Initialise charts on mount
  onMount(() => {
    // Initial calculation
    runSimulation();
    runPriceAnalysis();

    // Initialise Age Chart
    if (ageCanvasElement) {
      const ageCtx = ageCanvasElement.getContext('2d');
      ageChart = new Chart(ageCtx, {
        type: 'line',
        data: { labels: [], datasets: [] },
        options: {
          responsive: true,
          maintainAspectRatio: false,
          interaction: {
            intersect: false,
            mode: 'index'
          },
          scales: {
            x: {
              title: {
                display: true,
                text: 'Time (months)'
              }
            },
            y: {
              title: {
                display: true,
                text: 'Average Age (months)'
              }
            }
          },
          plugins: {
            legend: {
              display: true
            },
            tooltip: {
              mode: 'index',
              intersect: false
            }
          }
        }
      });
    }

    // Initialise Price Chart
    if (priceCanvasElement) {
      const priceCtx = priceCanvasElement.getContext('2d');
      priceChart = new Chart(priceCtx, {
        type: 'line',
        data: { datasets: [] },
        options: {
          responsive: true,
          maintainAspectRatio: false,
          interaction: {
            intersect: false,
            mode: 'index'
          },
          scales: {
            x: {
              type: 'linear',
              title: {
                display: true,
                text: 'Price Multiplier (1.0 = Current Price)'
              },
              min: 0.5,
              max: 1.5
            },
            y: {
              title: {
                display: true,
                text: 'Total Revenue per Cycle ($)'
              }
            }
          },
          plugins: {
            legend: {
              display: true
            },
            tooltip: {
              mode: 'index',
              intersect: false
            }
          }
        }
      });
    }

    // Initial chart update
    setTimeout(() => {
      updateCharts();
    }, 100);

    return () => {
      if (updateTimeout) {
        clearTimeout(updateTimeout);
      }
      if (ageChart) ageChart.destroy();
      if (priceChart) priceChart.destroy();
    };
  });

  // REMOVED THE $effect BLOCK THAT WAS CAUSING INFINITE LOOPS
</script>

<div class="min-h-screen bg-gradient-to-br from-slate-50 via-stone-50 to-amber-50">
  <!-- Header -->
  <div class="bg-gradient-to-r from-slate-900 via-stone-900 to-amber-900 text-white">
    <div class="max-w-7xl mx-auto px-6 py-8">
      <div class="flex items-center justify-between">
        <div>
          <h1 class="text-4xl font-light tracking-wide">Solera Analytics</h1>
          <p class="text-amber-200 mt-2 font-light">Professional Aging System Simulation & Revenue Optimisation</p>
        </div>
        <div class="text-right">
          <div class="text-sm text-amber-200 uppercase tracking-widest">Estate Management</div>
          <div class="text-2xl font-light">Solera Suite</div>
        </div>
      </div>
    </div>
  </div>

  <div class="max-w-7xl mx-auto px-6 py-8">
    <!-- System Configuration Panel -->
    <div class="bg-white/80 backdrop-blur-sm border border-amber-200/50 rounded-xl shadow-xl mb-8">
      <div class="bg-gradient-to-r from-amber-50 to-stone-50 px-6 py-4 border-b border-amber-200/50 rounded-t-xl">
        <h2 class="text-2xl font-light text-slate-800">System Configuration</h2>
        <p class="text-slate-600 text-sm mt-1">Configure your solera parameters for optimal aging and revenue performance</p>
      </div>
      
      <div class="p-6">
        <!-- Core Parameters -->
        <div class="grid grid-cols-2 md:grid-cols-4 gap-6 mb-8">
          <div class="group">
            <label for="numBarrels" class="block text-sm font-medium text-slate-700 mb-2 group-hover:text-amber-700 transition-colors">Barrel Count</label>
            <input
              id="numBarrels"
              type="number"
              min="1"
              max="10"
              bind:value={params.numBarrels}
              oninput={(e) => handleParamChange('numBarrels', e.target.value)}
              class="w-full px-4 py-3 border border-slate-300 rounded-lg focus:ring-2 focus:ring-amber-500 focus:border-amber-500 transition-all bg-white/50"
            />
          </div>
          <div class="group">
            <label for="inputVolume" class="block text-sm font-medium text-slate-700 mb-2 group-hover:text-amber-700 transition-colors">Input Volume <span class="text-slate-500">(ml/cycle)</span></label>
            <input
              id="inputVolume"
              type="number"
              bind:value={params.inputVolume}
              oninput={(e) => handleParamChange('inputVolume', e.target.value)}
              class="w-full px-4 py-3 border border-slate-300 rounded-lg focus:ring-2 focus:ring-amber-500 focus:border-amber-500 transition-all bg-white/50"
            />
          </div>
          <div class="group">
            <label for="cycleLength" class="block text-sm font-medium text-slate-700 mb-2 group-hover:text-amber-700 transition-colors">Cycle Length <span class="text-slate-500">(months)</span></label>
            <input
              id="cycleLength"
              type="number"
              step="0.5"
              bind:value={params.cycleLength}
              oninput={(e) => handleParamChange('cycleLength', e.target.value)}
              class="w-full px-4 py-3 border border-slate-300 rounded-lg focus:ring-2 focus:ring-amber-500 focus:border-amber-500 transition-all bg-white/50"
            />
          </div>
          <div class="group">
            <label for="simulationMonths" class="block text-sm font-medium text-slate-700 mb-2 group-hover:text-amber-700 transition-colors">Simulation Period <span class="text-slate-500">(months)</span></label>
            <input
              id="simulationMonths"
              type="number"
              bind:value={params.simulationMonths}
              oninput={(e) => handleParamChange('simulationMonths', e.target.value)}
              class="w-full px-4 py-3 border border-slate-300 rounded-lg focus:ring-2 focus:ring-amber-500 focus:border-amber-500 transition-all bg-white/50"
            />
          </div>
        </div>
        
        <!-- Flow Analysis -->
        <div class="mb-8 bg-gradient-to-r from-amber-50/50 to-stone-50/50 rounded-lg border border-amber-200/30 p-6">
          <h3 class="text-lg font-medium text-slate-800 mb-4 flex items-center">
            <div class="w-2 h-2 bg-amber-500 rounded-full mr-3"></div>
            Flow Analysis
          </h3>
          <div class="grid grid-cols-1 md:grid-cols-2 gap-6">
            <div>
              <h4 class="font-medium text-slate-700 mb-3">Volume Flow <span class="text-slate-500 font-normal">(ml/cycle)</span></h4>
              <div class="space-y-2 text-sm">
                <div class="flex justify-between py-1">
                  <span class="text-slate-600">Fresh Input → Barrel 1:</span>
                  <span class="font-semibold text-slate-800">{params.inputVolume}ml</span>
                </div>
                {#each params.barrels as barrel, index}
                  {@const netTransfer = index === params.numBarrels - 1 ? 0 : 
                    Math.max(0, params.inputVolume - params.barrels.slice(0, index + 1).reduce((sum, b) => sum + b.drainVolume, 0))}
                  <div class="flex justify-between py-1 border-t border-amber-200/30">
                    <span class="text-slate-600">
                      Barrel {index + 1} → {index === params.numBarrels - 1 ? 'Output' : `Barrel ${index + 2}`}:
                    </span>
                    <span class="font-semibold text-slate-800">
                      {index === params.numBarrels - 1 ? `${barrel.drainVolume}ml` : `${netTransfer}ml`}
                    </span>
                  </div>
                {/each}
              </div>
            </div>
            <div>
              <h4 class="font-medium text-slate-700 mb-3">Turnover Rates <span class="text-slate-500 font-normal">(%/cycle)</span></h4>
              <div class="space-y-2 text-sm">
                {#each params.barrels as barrel, index}
                  {@const turnoverRate = (barrel.drainVolume / barrel.capacity * 100)}
                  <div class="flex justify-between py-1">
                    <span class="text-slate-600">Barrel {index + 1}:</span>
                    <div class="text-right">
                      <span class="font-semibold text-slate-800">{turnoverRate.toFixed(1)}%</span>
                      <div class="text-xs text-slate-500">({barrel.drainVolume}ml / {barrel.capacity}ml)</div>
                    </div>
                  </div>
                {/each}
              </div>
            </div>
          </div>
          
          <!-- System Balance -->
          <div class="mt-4 p-4 bg-white/60 rounded-lg border border-amber-200/50">
            <h5 class="font-medium text-slate-700 mb-2">System Balance</h5>
            <div class="text-sm">
              <span class="font-medium {flowBalance === 0 ? 'text-emerald-600' : flowBalance > 0 ? 'text-amber-600' : 'text-red-600'}">
                Input: {params.inputVolume}ml | Total Drain: {totalDrain}ml | Balance: {flowBalance > 0 ? '+' : ''}{flowBalance}ml
                {#if flowBalance !== 0}
                  <span class="ml-2 font-normal">
                    {flowBalance > 0 ? '(System will overflow)' : '(System will empty)'}
                  </span>
                {/if}
              </span>
            </div>
          </div>
        </div>
        
        <!-- Barrel Configuration -->
        <div>
          <h3 class="text-lg font-medium text-slate-800 mb-4 flex items-center">
            <div class="w-2 h-2 bg-amber-500 rounded-full mr-3"></div>
            Barrel Configuration
          </h3>
          <div class="grid gap-4">
            {#each params.barrels as barrel, index}
              <div class="bg-white/60 rounded-lg border border-slate-200 p-4 hover:shadow-md transition-all">
                <div class="flex items-center mb-3">
                  <div 
                    class="w-4 h-4 rounded-full mr-3 shadow-sm" 
                    style="background-color: {barrelColors[index]}"
                  ></div>
                  <h4 class="font-medium text-slate-800">
                    Barrel {index + 1} 
                    <span class="ml-2 text-sm font-normal text-slate-500">
                      {index === 0 ? '(Youngest)' : index === params.numBarrels - 1 ? '(Oldest)' : '(Intermediate)'}
                    </span>
                  </h4>
                </div>
                <div class="grid grid-cols-3 gap-4">
                  <div>
                    <label for={`capacity-${index}`} class="block text-xs font-medium text-slate-600 mb-1">Capacity (ml)</label>
                    <input
                      id={`capacity-${index}`}
                      type="number"
                      bind:value={barrel.capacity}
                      oninput={(e) => handleBarrelParamChange(index, 'capacity', e.target.value)}
                      class="w-full px-3 py-2 text-sm border border-slate-300 rounded-md focus:ring-2 focus:ring-amber-400 focus:border-amber-400 transition-all bg-white/70"
                    />
                  </div>
                  <div>
                    <label for={`drainVolume-${index}`} class="block text-xs font-medium text-slate-600 mb-1">Drain Volume (ml/cycle)</label>
                    <input
                      id={`drainVolume-${index}`}
                      type="number"
                      bind:value={barrel.drainVolume}
                      oninput={(e) => handleBarrelParamChange(index, 'drainVolume', e.target.value)}
                      class="w-full px-3 py-2 text-sm border border-slate-300 rounded-md focus:ring-2 focus:ring-amber-400 focus:border-amber-400 transition-all bg-white/70"
                    />
                  </div>
                  <div>
                    <label for={`pricePerLiter-${index}`} class="block text-xs font-medium text-slate-600 mb-1">Price ($/L)</label>
                    <input
                      id={`pricePerLiter-${index}`}
                      type="number"
                      step="0.01"
                      bind:value={barrel.pricePerLiter}
                      oninput={(e) => handleBarrelParamChange(index, 'pricePerLiter', e.target.value)}
                      class="w-full px-3 py-2 text-sm border border-slate-300 rounded-md focus:ring-2 focus:ring-amber-400 focus:border-amber-400 transition-all bg-white/70"
                    />
                  </div>
                </div>
              </div>
            {/each}
          </div>
        </div>
      </div>
    </div>

    <!-- Analytics Dashboard -->
    <div class="grid grid-cols-1 lg:grid-cols-2 gap-8 mb-8">
      <!-- Age Analysis Chart -->
      <div class="bg-white/80 backdrop-blur-sm border border-amber-200/50 rounded-xl shadow-xl overflow-hidden">
        <div class="bg-gradient-to-r from-slate-50 to-stone-50 px-6 py-4 border-b border-slate-200">
          <h2 class="text-xl font-medium text-slate-800">Aging Profile Analysis</h2>
          <p class="text-slate-600 text-sm mt-1">Average age progression by barrel position</p>
        </div>
        <div class="p-6">
          <div style="position: relative; height: 400px; width: 100%;">
            <canvas bind:this={ageCanvasElement}></canvas>
          </div>
        </div>
      </div>

      <!-- Price Sensitivity Chart -->
      <div class="bg-white/80 backdrop-blur-sm border border-amber-200/50 rounded-xl shadow-xl overflow-hidden">
        <div class="bg-gradient-to-r from-slate-50 to-stone-50 px-6 py-4 border-b border-slate-200">
          <h2 class="text-xl font-medium text-slate-800">Price Sensitivity Matrix</h2>
          <p class="text-slate-600 text-sm mt-1">Revenue impact of pricing adjustments by barrel</p>
        </div>
        <div class="p-6">
          <div style="position: relative; height: 400px; width: 100%;">
            <canvas bind:this={priceCanvasElement}></canvas>
          </div>
        </div>
      </div>
    </div>

    <!-- Performance Summary -->
    {#if data.length > 0}
      <div class="bg-white/80 backdrop-blur-sm border border-amber-200/50 rounded-xl shadow-xl">
        <div class="bg-gradient-to-r from-amber-50 to-stone-50 px-6 py-4 border-b border-amber-200/50">
          <h2 class="text-2xl font-medium text-slate-800">Executive Performance Summary</h2>
          <p class="text-slate-600 text-sm mt-1">Comprehensive system analysis and financial metrics</p>
        </div>
        
        <div class="p-6">
          <!-- Barrel Performance Grid -->
          <div class="mb-8">
            <h3 class="text-lg font-medium text-slate-800 mb-4 flex items-center">
              <div class="w-2 h-2 bg-amber-500 rounded-full mr-3"></div>
              Barrel Performance Analysis
            </h3>
            <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4">
              {#each Array.from({ length: params.numBarrels }) as _, index}
                <div class="bg-gradient-to-br from-white to-stone-50 rounded-lg border border-slate-200 p-4 hover:shadow-md transition-all">
                  <div class="flex items-center mb-3">
                    <div 
                      class="w-4 h-4 rounded-full mr-3 shadow-sm" 
                      style="background-color: {barrelColors[index]}"
                    ></div>
                    <div class="font-medium text-slate-800">Barrel {index + 1}</div>
                  </div>
                  <div class="space-y-2 text-sm">
                    <div class="flex justify-between">
                      <span class="text-slate-600">Age:</span>
                      <span class="font-semibold text-slate-800">{latestData?.[`barrel${index}AvgAge`]?.toFixed(1) || '0.0'} months</span>
                    </div>
                    <div class="flex justify-between">
                      <span class="text-slate-600">Revenue/cycle:</span>
                      <span class="font-semibold text-emerald-600">${latestData?.[`barrel${index}Revenue`]?.toFixed(2) || '0.00'}</span>
                    </div>
                    <div class="flex justify-between">
                      <span class="text-slate-600">Total revenue:</span>
                      <span class="font-semibold text-emerald-700">${latestData?.[`barrel${index}CumulativeRevenue`]?.toFixed(2) || '0.00'}</span>
                    </div>
                    <div class="pt-2 border-t border-slate-200">
                      <div class="flex justify-between text-xs">
                        <span class="text-slate-500">Price:</span>
                        <span class="text-slate-700">${params.barrels[index]?.pricePerLiter}/L</span>
                      </div>
                      <div class="flex justify-between text-xs">
                        <span class="text-slate-500">Drain:</span>
                        <span class="text-slate-700">{params.barrels[index]?.drainVolume}ml ({(params.barrels[index]?.drainVolume / 1000).toFixed(2)}L)</span>
                      </div>
                      <div class="flex justify-between text-xs text-slate-500">
                        <span>Check:</span>
                        <span>${((params.barrels[index]?.drainVolume / 1000) * params.barrels[index]?.pricePerLiter).toFixed(2)}</span>
                      </div>
                    </div>
                  </div>
                </div>
              {/each}
            </div>
          </div>
          
          <!-- Key Metrics -->
          <div class="grid grid-cols-2 md:grid-cols-4 gap-6 mb-6">
            <div class="bg-gradient-to-br from-emerald-50 to-emerald-100 rounded-lg p-4 border border-emerald-200">
              <div class="text-sm font-medium text-emerald-700 mb-1">Total Revenue</div>
              <div class="text-2xl font-bold text-emerald-800">${latestData?.totalCumulativeRevenue.toFixed(0) || '0'}</div>
              <div class="text-xs text-emerald-600 mt-1">Cumulative</div>
            </div>
            <div class="bg-gradient-to-br from-amber-50 to-amber-100 rounded-lg p-4 border border-amber-200">
              <div class="text-sm font-medium text-amber-700 mb-1">Revenue/Month</div>
              <div class="text-2xl font-bold text-amber-800">${latestData?.valuePerMonth.toFixed(0) || '0'}</div>
              <div class="text-xs text-amber-600 mt-1">Average</div>
            </div>
            <div class="bg-gradient-to-br from-blue-50 to-blue-100 rounded-lg p-4 border border-blue-200">
              <div class="text-sm font-medium text-blue-700 mb-1">System Efficiency</div>
              <div class="text-2xl font-bold text-blue-800">{systemEfficiency}%</div>
              <div class="text-xs text-blue-600 mt-1">Extraction Rate</div>
            </div>
            <div class="bg-gradient-to-br from-purple-50 to-purple-100 rounded-lg p-4 border border-purple-200">
              <div class="text-sm font-medium text-purple-700 mb-1">Age Premium</div>
              <div class="text-2xl font-bold text-purple-800">{agePremium}x</div>
              <div class="text-xs text-purple-600 mt-1">Oldest vs Youngest</div>
            </div>
          </div>
          
          <!-- System Description -->
          <div class="bg-gradient-to-r from-slate-50 to-stone-50 rounded-lg p-6 border border-slate-200">
            <h4 class="font-medium text-slate-800 mb-2">System Operation</h4>
            <p class="text-sm text-slate-600 leading-relaxed">
              Fresh liquid ({params.inputVolume}ml) enters Barrel 1 every {params.cycleLength} months. Each barrel drains its specified volume at individual pricing tiers, with liquid cascading through the system using FIFO aging methodology. Premium pricing reflects extended maturation periods, creating a sophisticated revenue optimisation matrix for estate-level production planning.
            </p>
          </div>
        </div>
      </div>
    {/if}
  </div>
</div>
