


<template>
    <details open class="bulk-search-controls">
        <summary>Input</summary>
        <div>
            <p v-if="showColumnHint">Please include a header row. Supported columns include: "Title", "Author".</p>
            <textarea v-model="bulkSearchState.inputText"></textarea>
            <br />
            <label>Format: <select v-model="bulkSearchState._activeExtractorIndex">
                    <option v-for="extractor, index in bulkSearchState.extractors" :value = "index" :key="index">
                        {{ extractor.label }}
                    </option>
                </select></label>
            <label v-if="this.showApiKey">OpenAI API Key:
                <input v-if="showPassword" type="password" @click="togglePasswordVisibility()" v-model="bulkSearchState.extractionOptions.openaiApiKey" />

                <input v-else type="text" @click="togglePasswordVisibility" v-model="bulkSearchState.extractionOptions.openaiApiKey" />
            </label>
            <label>Sample Data:
                <select v-model="selectedValue">
                    <option v-for="sample in sampleData" :value = "sample.text" :key = "sample.source"> {{sample.name}} </option>
                </select>
            </label>
            <label>
                <input v-model="bulkSearchState.matchOptions.includeAuthor" type="checkbox" /> Use author in
                search query
            </label>
            <br>
            <button @click="extractBooks" :disabled="loadingExtractedBooks">{{ extractBooksText }}</button>
            <button @click="matchBooks" :disabled="loadingMatchedBooks">{{ matchBooksText }}</button>
        </div>
        <div v-if="bulkSearchState.errorMessage">
            <p v-for="error in bulkSearchState.errorMessage" :key="error">
                {{ error }}</p>
        </div>
    </details>
</template>

<script>
import {sampleData} from '../utils/samples.js';
import { BulkSearchState} from '../utils/classes.js';
import { buildSearchUrl } from '../utils/searchUtils.js'
export default {

    props: {
        bulkSearchState: BulkSearchState
    },
    data() {
        return {
            selectedValue: '',
            showPassword: true,
            sampleData: sampleData,
            loadingExtractedBooks: false,
            loadingMatchedBooks: false,
        }
    },
    watch: {
        selectedValue(newValue) {
            if (newValue!==''){
                this.bulkSearchState.inputText = newValue;
            }
        }
    },
    computed: {
        showApiKey(){
            if (this.bulkSearchState.activeExtractor) return 'model' in this.bulkSearchState.activeExtractor
            return false
        },
        extractBooksText(){
            if (this.loadingExtractedBooks) return 'Loading...'
            return 'Extract Books'
        },
        matchBooksText(){
            if (this.loadingMatchedBooks) return 'Loading...'
            return 'Match Books'
        },
        showColumnHint(){
            if (this.bulkSearchState.activeExtractor) return this.bulkSearchState.activeExtractor.name === 'table_extractor'
            return false
        },
    },
    methods: {
        togglePasswordVisibility(){
            this.showPassword= !this.showPassword
        },
        async extractBooks() {
            this.loadingExtractedBooks = true
            const extractedData = await this.bulkSearchState.activeExtractor.run(this.bulkSearchState.extractionOptions, this.bulkSearchState.inputText)
            this.bulkSearchState.matchedBooks = extractedData
            this.loadingExtractedBooks = false
        },
        async matchBooks() {
            const fetchSolrBook = async function (book, matchOptions) {
                try {
                    const data = await fetch(buildSearchUrl(book, matchOptions, true))
                    return await data.json()
                }
                catch (error) {}
            }
            this.loadingMatchedBooks = true
            for (const bookMatch of this.bulkSearchState.matchedBooks) {
                bookMatch.solrDocs = await fetchSolrBook(bookMatch.extractedBook, this.bulkSearchState.matchOptions)
            }
            this.loadingMatchedBooks = false
        },

    }
}
</script>

<style lang="less">

.bulk-search-controls{
    padding:20px;
}
label input {
    flex: 1;
}

textarea {
    width: 100%;
    display: flex;
    resize: vertical;
    box-sizing: border-box;
}
</style>
