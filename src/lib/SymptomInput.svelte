<script>
    import { Mic, MicOff } from 'lucide-svelte';
  
    let isListening = false;
    let recognition;
    let symptomText = '';
    let age = '';
    let country = 'Nigeria';
    let gender = '';
    let diagnosisResults = [];
    let isLoading = false;
    let errorMessage = '';
    let cachedResponse = null;
    let hasSubmitted = false;
    
    function toggleSpeechRecognition() {
        // @ts-ignore
        const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
        
        if (!SpeechRecognition) {
            alert('Speech recognition not supported in your browser');
            return;
        }
        
        if (!recognition) {
        recognition = new SpeechRecognition();
        recognition.continuous = true;
        recognition.interimResults = true;
        
        recognition.onresult = (event) => {
            const transcript = Array.from(event.results)
            .map(result => result[0])
            .map(result => result.transcript)
            .join('');
            symptomText = transcript;
        };
        
        recognition.onerror = (event) => {
            console.error('Speech recognition error', event.error);
            isListening = false;
        };
        
        recognition.onend = () => {
            isListening = false;
        };
        }
        
        if (isListening) {
            recognition.stop();
            isListening = false;
        } else {
            recognition.start();
            isListening = true;
        }
    }

    function extractXmlFromReport(reportString) {
        try {
            const xmlStart = reportString.indexOf('<diseases>');
            const xmlEnd = reportString.indexOf('</diseases>') + '</diseases>'.length;
            return reportString.slice(xmlStart, xmlEnd);
        } catch (e) {
            console.error('XML extraction error:', e);
            return '';
        }
    }


    function parseXmlResults(xmlString) {
        try {
            const parser = new DOMParser();
            const xmlDoc = parser.parseFromString(xmlString, "text/xml");
            const diseases = xmlDoc.getElementsByTagName("disease");
            
            return Array.from(diseases).map(disease => ({
                name: disease.getElementsByTagName("name")[0]?.textContent || 'Unknown',
                background: disease.getElementsByTagName("background")[0]?.textContent || '',
                symptoms: disease.getElementsByTagName("symptoms")[0]?.textContent || '',
                transmission: disease.getElementsByTagName("transmission")[0]?.textContent || '',
                testing: disease.getElementsByTagName("testing")[0]?.textContent || '',
                treatment: disease.getElementsByTagName("treatment")[0]?.textContent || '',
                reference: disease.getElementsByTagName("reference")[0]?.textContent || ''
            }));
        } catch (e) {
            console.error('XML parsing error:', e);
            return [];
        }
    }



    async function handleSubmit(event) {
        event.preventDefault();
        hasSubmitted = true;
        isLoading = true;
        errorMessage = '';
        
        // Create cache key based on inputs
        const cacheKey = `${age}-${country}-${gender}-${symptomText}`;
        
        // Check cache first
        if (cachedResponse && cachedResponse.key === cacheKey) {
            diagnosisResults = cachedResponse.data;
            isLoading = false;
            return;
        }

        try {
            const demographicString = `${age}, ${country}, ${gender}`;
            const response = await fetch('https://5000-firebase-clinicheck-backend-1747325589411.cluster-c23mj7ubf5fxwq6nrbev4ugaxa.cloudworkstations.dev/analyse-symptoms', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({
                    demographics: demographicString,
                    symptoms: symptomText
                })
            });

            if (!response.ok) throw new Error(`HTTP error! status: ${response.status}`);
            
            const data = await response.json();
            const xmlString = extractXmlFromReport(data.report);
            diagnosisResults = parseXmlResults(xmlString);
            
            // Cache the response
            cachedResponse = {
                key: cacheKey,
                data: diagnosisResults
            };
            
        } catch (error) {
            console.error('API Error:', error);
            errorMessage = 'Failed to get diagnosis. Please try again.';
            diagnosisResults = [];
        } finally {
            isLoading = false;
        }
    }
  
</script>

<div class="flex flex-col items-center min-h-screen bg-gray-50 p-4 pt-8">
    <form on:submit|preventDefault={handleSubmit} class="w-full max-w-md p-8 bg-white rounded-lg shadow-sm border border-gray-200 mb-6">
        <h2 class="text-2xl font-bold text-gray-800 mb-6 text-left">Clinicheck</h2>
        <div class="mb-4">
            <label for="age" class="block text-gray-700 font-medium mb-2">Age</label>
            <input 
            type="text" 
            id="age"
            bind:value={age} 
            name="age" 
            placeholder="e.g, 35"
            class="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:border-gray-500 text-gray-800"
            >
        </div>
      
        <div class="mb-4">
            <label for="country" class="block text-gray-700 font-medium mb-2">Country of Residence</label>
            <input 
            type="text" 
            id="country"
            bind:value={country} 
            name="country" 
            placeholder="e.g, Nigeria"
            class="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:border-gray-500 text-gray-800"
            >
        </div>
      
        <div class="mb-4">
            <label for="gender" class="block text-gray-700 font-medium mb-2">Gender at Birth</label>
            <select
            id="gender"
            bind:value={gender}
            name="gender"
            class="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:border-gray-500 text-gray-800"
            >
            <option value="" selected disabled>--Select--</option>
            <option value="male">Male</option>
            <option value="female">Female</option>
            </select>
        </div>
      
        <div class="mb-6">
            <label for="symptom" class="block text-gray-700 font-medium mb-2">Symptom</label>
            <div class="relative">
              <input 
                type="text" 
                id="symptom" 
                name="symptom" 
                placeholder="Describe your symptoms in a few words"
                bind:value={symptomText}
                class="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:border-gray-500 text-gray-800 pr-10"
              >
              <button
                type="button"
                on:click={toggleSpeechRecognition}
                class="absolute inset-y-0 right-0 flex items-center pr-3 {isListening ? 'bg-red-100' : ''}"
              >
                {#if isListening}
                  <MicOff class="w-5 h-5 text-gray-600 hover:text-gray-800" />
                {:else}
                  <Mic class="w-5 h-5 text-gray-600 hover:text-gray-800" />
                {/if}
              </button>
            </div>
            {#if isListening}
              <p class="mt-2 text-sm text-gray-500">Listening... Speak now</p>
            {/if}
          </div>
      
        <button 
            type="submit" 
            class="flex justify-end bg-gray-800 text-white py-2 px-4 rounded-md hover:bg-gray-700 focus:outline-none focus:ring-2 focus:ring-gray-500 focus:ring-offset-2 disabled:opacity-50"
            disabled={isLoading}>
            {isLoading ? 'Analyzing...' : 'Analyze Symptoms'}
        </button>
    </form>

    <!-- Results Display Section -->
    <div class="w-full max-w-md flex-1 overflow-y-auto">
        {#if isLoading}
            <div class="text-center py-4">
                <p class="text-gray-600">Analyzing symptoms...</p>
            </div>
        {:else if errorMessage}
            <div class="p-4 bg-red-50 rounded-lg border border-red-200 mb-6">
                <p class="text-red-600">{errorMessage}</p>
            </div>
        {:else if diagnosisResults.length > 0}
            <div class="mb-6">
                <h3 class="text-xl font-semibold text-gray-800 mb-4">Diagnosis Results</h3>
                <div class="space-y-4">
                    {#each diagnosisResults as disease}
                        <div class="p-6 bg-white rounded-lg shadow-sm border border-gray-200">
                            <h4 class="text-lg font-medium text-gray-800 mb-2">{disease.name}</h4>
                            
                            <div class="space-y-3">
                                {#if disease.background}
                                    <div>
                                        <h5 class="font-medium text-gray-700">Background</h5>
                                        <p class="text-gray-600">{disease.background}</p>
                                    </div>
                                {/if}
                                
                                {#if disease.symptoms}
                                    <div>
                                        <h5 class="font-medium text-gray-700">Symptoms</h5>
                                        <p class="text-gray-600">{disease.symptoms}</p>
                                    </div>
                                {/if}
                                
                                {#if disease.transmission}
                                    <div>
                                        <h5 class="font-medium text-gray-700">Transmission</h5>
                                        <p class="text-gray-600">{disease.transmission}</p>
                                    </div>
                                {/if}
                                
                                {#if disease.testing}
                                    <div>
                                        <h5 class="font-medium text-gray-700">Testing</h5>
                                        <p class="text-gray-600">{disease.testing}</p>
                                    </div>
                                {/if}
                                
                                {#if disease.treatment}
                                    <div>
                                        <h5 class="font-medium text-gray-700">Treatment</h5>
                                        <p class="text-gray-600">{disease.treatment}</p>
                                    </div>
                                {/if}
                                
                                {#if disease.reference}
                                    <div>
                                        <h5 class="font-medium text-gray-700">Reference</h5>
                                        <p class="text-gray-600 text-sm">{disease.reference}</p>
                                    </div>
                                {/if}
                            </div>
                        </div>
                    {/each}
                </div>
            </div>
        {:else if hasSubmitted && !isLoading}
            <div class="p-6 bg-white rounded-lg shadow-sm border border-gray-200 text-center">
                <p class="font-bold text-gray-800">No diagnosis found</p>
                <p class="text-gray-600 mt-2">We couldn't find any matching diagnoses for your symptoms.</p>
            </div>
        {/if}
    </div>
</div>