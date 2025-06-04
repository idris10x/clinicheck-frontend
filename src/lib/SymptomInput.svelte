<script>
    import { Mic, MicOff } from 'lucide-svelte';

    let isListening = $state(false);
    let recognition;
    let symptomText = $state('');
    let age = $state('');
    let country = $state('Nigeria');
    let gender = $state('');
    let diagnosisResults = $state([]);
    let drugInteractionResults = $state([]); 
    let isLoading = $state(false);
    let errorMessage = $state('');
    let cachedResponse = $state(null);
    let hasSubmitted = $state(false);
    let activeTab = $state('diagnosis');

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

    
    function parseDiseaseXmlResults(xmlString) {
        if (!xmlString || xmlString.trim() === '') {
            return [];
        }
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
            console.error('Disease XML parsing error:', e);
            return [];
        }
    }

    
    function parseDrugXmlResults(xmlString) {
        if (!xmlString || xmlString.trim() === '') {
            return [];
        }
        try {
            const parser = new DOMParser();
            const xmlDoc = parser.parseFromString(xmlString, "text/xml");
            const drugReportNode = xmlDoc.getElementsByTagName("drugreport")[0];
            if (!drugReportNode) return [];

            const drugs = [];
            const children = drugReportNode.childNodes;
            let currentDrug = null;

            for (let i = 0; i < children.length; i++) {
                const node = children[i];
                // Ensure it's an element node (nodeType 1)
                if (node.nodeType === 1) { 
                    if (node.nodeName === "drugname") {
                        if (currentDrug) { // Push the previously collected drug
                            drugs.push(currentDrug);
                        }
                        currentDrug = {
                            name: node.textContent || 'Unknown Drug',
                            adverseReactions: '',
                            indicationsForUse: ''
                        };
                    } else if (currentDrug) {
                        if (node.nodeName === "adversereactions") {
                            currentDrug.adverseReactions = node.textContent || '';
                        } else if (node.nodeName === "indicationsforuse") {
                            currentDrug.indicationsForUse = node.textContent || '';
                        }
                    }
                }
            }
            if (currentDrug) { // Push the last collected drug
                drugs.push(currentDrug);
            }
            return drugs;
        } catch (e) {
            console.error('Drug XML parsing error:', e);
            return [];
        }
    }

    async function handleSubmit(event) {
        event.preventDefault();
        hasSubmitted = true;
        isLoading = true;
        errorMessage = '';
        diagnosisResults = [];      
        drugInteractionResults = [];
        activeTab = 'diagnosis'; 
        
        const cacheKey = `${age}-${country}-${gender}-${symptomText}`;
        
        if (cachedResponse && cachedResponse.key === cacheKey) {
            diagnosisResults = cachedResponse.diagnosisData;
            drugInteractionResults = cachedResponse.drugData;
            isLoading = false;
            return;
        }

        try {
            const demographicString = `${age}, ${country}, ${gender}`;
            const response = await fetch(import.meta.env.VITE_SYMPTOM_API + '/analyse-symptoms', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({
                    demographics: demographicString,
                    symptoms: symptomText
                })
            });

            if (!response.ok) throw new Error(`HTTP error! status: ${response.status}`);
            
            const data = await response.json();

            if (!data.report) {
                throw new Error('API response missing "report" object');
            }
            
            const diseaseXmlString = data.report.disease_report_xml || "";
            const drugXmlString = data.report.drug_report_xml || "";
            
            diagnosisResults = parseDiseaseXmlResults(diseaseXmlString);
            drugInteractionResults = parseDrugXmlResults(drugXmlString);
            
            cachedResponse = {
                key: cacheKey,
                diagnosisData: diagnosisResults,
                drugData: drugInteractionResults
            };
            
        } catch (error) {
            console.error('API Error:', error);
            errorMessage = `Failed to get results: ${error.message}. Please try again.`;
            diagnosisResults = [];
            drugInteractionResults = [];
        } finally {
            isLoading = false;
        }
    }
</script>

<div class="flex flex-col items-center min-h-screen bg-gray-50 p-4 pt-8">
    <form onsubmit={handleSubmit} class="w-full max-w-md p-8 bg-white rounded-[20px] shadow-xl border border-gray-200 mb-6">
        <h2 class="text-2xl font-bold text-[#3E73A6] mb-6 text-left">Clinicheck</h2>
        <div class="mb-4">
            <label for="age" class="block text-[#3E73A6] font-semibold mb-2">Age</label>
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
            <label for="country" class="block text-[#3E73A6] font-semibold mb-2">Country of Residence</label>
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
            <label for="gender" class="block text-[#3E73A6] font-semibold mb-2">Gender at Birth</label>
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
            <label for="symptom" class="block text-[#3E73A6] font-semibold mb-2">Symptom</label>
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
                    onclick={toggleSpeechRecognition}
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
            class="flex justify-end bg-[#3E73A6] text-white font-semibold py-2 px-4 rounded-md hover:bg-[#2E5A8C] focus:outline-none focus:ring-2 focus:ring-gray-500 focus:ring-offset-2 disabled:opacity-50"
            disabled={isLoading}>
            {isLoading ? 'Analyzing...' : 'Analyze Symptoms'}
        </button>
    </form>

    <div class="w-full max-w-md flex-1 overflow-y-auto">
        {#if isLoading}
            <div class="text-center py-4">
                <p class="text-gray-600">Analyzing symptoms...</p>
            </div>
        {:else if errorMessage}
            <div class="p-4 bg-red-50 rounded-lg border border-red-200 mb-6">
                <p class="text-red-600">{errorMessage}</p>
            </div>
        {:else if hasSubmitted} <div class="mb-1">
                <div class="flex">
                    <button
                        type="button"
                        class="flex-1 text-center py-3 px-2 focus:outline-none relative"
                        onclick={() => activeTab = 'diagnosis'}
                    >
                        <h3 class="text-lg font-semibold {activeTab === 'diagnosis' ? 'text-[#3E73AC]' : 'text-gray-700'}">Diagnosis Result</h3>
                        {#if activeTab === 'diagnosis'}
                            <div class="absolute bottom-0 left-0 right-0 h-[3px] bg-[#3E73AC]"></div>
                        {/if}
                    </button>
                    <button
                        type="button"
                        class="flex-1 text-center py-3 px-2 focus:outline-none relative"
                        onclick={() => activeTab = 'drugInteractions'}
                    >
                        <h3 class="text-lg font-semibold {activeTab === 'drugInteractions' ? 'text-[#3E73AC]' : 'text-gray-700'}">Drug Interactions</h3>
                        {#if activeTab === 'drugInteractions'}
                            <div class="absolute bottom-0 left-0 right-0 h-[3px] bg-[#3E73AC]"></div>
                        {/if}
                    </button>
                </div>
                <div class="w-full h-px bg-[#B8B8B8]"></div> </div>

            <div class="pt-4">
                {#if activeTab === 'diagnosis'}
                    {#if diagnosisResults.length > 0}
                        <div class="space-y-4">
                            {#each diagnosisResults as disease}
                                <div class="p-6 bg-[#256099] rounded-[20px] shadow-xl border border-gray-200">
                                    <h4 class="text-lg font-bold text-[#FDFDFD] mb-3">{disease.name}</h4>
                                    <div class="space-y-3">
                                        {#if disease.background}
                                            <div>
                                                <h5 class="font-semibold text-[#B7CEE5]">Background</h5>
                                                <p class="text-[#FDFDFD] text-sm">{disease.background}</p>
                                            </div>
                                        {/if}
                                        {#if disease.symptoms}
                                            <div>
                                                <h5 class="font-semibold text-[#B7CEE5]">Symptoms</h5>
                                                <p class="text-[#FDFDFD] text-sm">{disease.symptoms}</p>
                                            </div>
                                        {/if}
                                        {#if disease.transmission}
                                            <div>
                                                <h5 class="font-semibold text-[#B7CEE5]">Transmission</h5>
                                                <p class="text-[#FDFDFD] text-sm">{disease.transmission}</p>
                                            </div>
                                        {/if}
                                        {#if disease.testing}
                                            <div>
                                                <h5 class="font-semibold text-[#B7CEE5]">Testing</h5>
                                                <p class="text-[#FDFDFD] text-sm">{disease.testing}</p>
                                            </div>
                                        {/if}
                                        {#if disease.treatment}
                                            <div>
                                                <h5 class="font-semibold text-[#B7CEE5]">Treatment</h5>
                                                <p class="text-[#FDFDFD] text-sm">{disease.treatment}</p>
                                            </div>
                                        {/if}
                                        {#if disease.reference}
                                            <div>
                                                <h5 class="font-semibold text-[#B7CEE5]">Reference</h5>
                                                <p class="text-[#FDFDFD] text-xs italic">{disease.reference}</p>
                                            </div>
                                        {/if}
                                    </div>
                                </div>
                            {/each}
                        </div>
                    {:else} <div class="p-6 bg-white rounded-lg shadow-sm border border-gray-200 text-center">
                            <p class="font-bold text-gray-800">No diagnosis found</p>
                            <p class="text-gray-600 mt-2">We couldn't find any matching diagnoses for your symptoms.</p>
                        </div>
                    {/if}
                {:else if activeTab === 'drugInteractions'}
                    {#if drugInteractionResults.length > 0}
                        <div class="space-y-4">
                            {#each drugInteractionResults as drug}
                                <div class="p-6 bg-[#256099] rounded-[20px] shadow-xl border border-gray-200">
                                    <h4 class="text-lg font-bold text-[#FDFDFD] mb-3">{drug.name}</h4>
                                    <div class="space-y-3">
                                        {#if drug.adverseReactions}
                                            <div>
                                                <h5 class="font-semibold text-[#B7CEE5]">Adverse Reactions</h5>
                                                <p class="text-[#FDFDFD] text-sm">{drug.adverseReactions}</p>
                                            </div>
                                        {/if}
                                        {#if drug.indicationsForUse}
                                            <div>
                                                <h5 class="font-semibold text-[#B7CEE5]">Indications for Use</h5>
                                                <p class="text-[#FDFDFD] text-sm">{drug.indicationsForUse}</p>
                                            </div>
                                        {:else}
                                             <div>
                                                <h5 class="font-semibold text-[#B7CEE5]">Indications for Use</h5>
                                                <p class="text-[#FDFDFD] text-sm">N/A</p>
                                            </div>
                                        {/if}
                                    </div>
                                </div>
                            {/each}
                        </div>
                    {:else}
                        <div class="p-6 bg-white rounded-lg shadow-sm border border-gray-200 text-center">
                            <p class="font-bold text-gray-800">No drug interaction information found</p>
                            <p class="text-gray-600 mt-2">We couldn't find any drug interaction data for the provided symptoms.</p>
                        </div>
                    {/if}
                {/if}
            </div>
        {/if}
    </div>
</div>