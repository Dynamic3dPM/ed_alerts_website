<template>
    <v-container class="pa-6">
        <v-row justify="center">
            <v-col v-for="(pair, i) in forexPairs" :key="i" cols="12" sm="6" md="4" lg="3">
                <v-card class="pa-4" elevation="2" rounded="lg" :color="getCardColor(pair)" hover>
                    <v-row align="center" no-gutters>
                        <v-col cols="2">
                            <v-icon :color="getIconColor(pair)" icon="mdi-currency-usd" size="32" />
                        </v-col>
                        <v-col cols="8">
                            <v-card-title class="text-h6 font-weight-medium">
                                {{ pair.name }}
                            </v-card-title>
                        </v-col>
                        <v-col cols="2" class="text-right">
                            <v-chip :color="getStatusColor(pair)" size="small" variant="flat">
                                {{ getStatusText(pair) }}
                            </v-chip>
                        </v-col>
                    </v-row>

                    <v-divider class="my-3" />

                    <v-card-text class="pa-0">
                        <v-list dense>
                            <v-list-item>
                                <v-list-item-title class="text-body-1">
                                    Williams %R:
                                    <span class="font-weight-bold">
                                        {{ pair.currentWilliams !== null ? pair.currentWilliams : 'N/A' }}
                                    </span>
                                </v-list-item-title>
                            </v-list-item>
                            <v-list-item v-if="pair.crossTime">
                                <v-list-item-title class="text-body-2 text-grey-darken-1">
                                    Cross Time: {{ pair.crossTime }}
                                </v-list-item-title>
                            </v-list-item>
                            <v-list-item v-if="pair.alert">
                                <v-list-item-title class="text-body-2 text-info">
                                    Alert: {{ pair.alert }}
                                </v-list-item-title>
                            </v-list-item>
                            <v-list-item v-if="pair.error">
                                <v-list-item-title class="text-body-2 text-warning">
                                    Error: {{ pair.error }}
                                </v-list-item-title>
                            </v-list-item>
                        </v-list>
                    </v-card-text>
                </v-card>
            </v-col>
        </v-row>
    </v-container>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import alertSound from '@/assets/05749 Emergency alert - looping.wav'

const forexPairs = ref([
    { name: 'AUD/USD', currentWilliams: null, previousWilliams: null, crossTime: null, alert: null, lastDate: null, error: null },
    { name: 'EUR/USD', currentWilliams: null, previousWilliams: null, crossTime: null, alert: null, lastDate: null, error: null },
    { name: 'NZD/USD', currentWilliams: null, previousWilliams: null, crossTime: null, alert: null, lastDate: null, error: null },
    { name: 'USD/CAD', currentWilliams: null, previousWilliams: null, crossTime: null, alert: null, lastDate: null, error: null },
    { name: 'USD/CHF', currentWilliams: null, previousWilliams: null, crossTime: null, alert: null, lastDate: null, error: null },
    { name: 'USD/JPY', currentWilliams: null, previousWilliams: null, crossTime: null, alert: null, lastDate: null, error: null }
])

const audio = new Audio(alertSound)

const requestNotificationPermission = async () => {
    if ('Notification' in window && Notification.permission !== 'granted') {
        await Notification.requestPermission()
    }
}

const triggerAlert = (pairName, message) => {
    if ('Notification' in window && Notification.permission === 'granted') {
        new Notification(`${pairName} Alert`, {
            body: message,
            icon: 'https://cdn-icons-png.flaticon.com/512/1828/1828490.png'
        })
    }
    audio.loop = true
    audio.play().catch(error => console.error('Error playing audio:', error))
    setTimeout(() => {
        audio.pause()
        audio.currentTime = 0
    }, 10000)
}

const processData = (data, pairName) => {
    const pair = forexPairs.value.find(p => p.name === pairName)
    if (!pair) {
        console.error(`Pair ${pairName} not found in forexPairs`)
        return
    }

    pair.error = null
    pair.alert = null

    if (!data || !Array.isArray(data) || data.length === 0) {
        pair.error = 'No data received'
        console.warn(`No valid data for ${pairName}`)
        return
    }

    const latestData = data[data.length - 1]
    const previousData = data.length > 1 ? data[data.length - 2] : null

    if (!latestData || typeof latestData.WILLR === 'undefined') {
        pair.error = 'Invalid data format - WILLR missing'
        console.warn(`Invalid data format for ${pairName}:`, latestData)
        return
    }

    const currentDate = new Date(latestData.datetime).toDateString()
    console.log(`${pairName} - Latest WILLR: ${latestData.WILLR}, Date: ${currentDate}`)

    if (pair.lastDate && pair.lastDate !== currentDate) {
        pair.previousWilliams = pair.currentWilliams
    }
    pair.currentWilliams = latestData.WILLR
    pair.lastDate = currentDate

    if (pair.previousWilliams !== null && previousData) {
        const wasOutOfZone = pair.previousWilliams > -80 && pair.previousWilliams < -20 // Between -20 and -80
        const nowInZoneBullish = pair.currentWilliams >= -20 // Above -20
        const nowInZoneBearish = pair.currentWilliams <= -80 // Below -80

        if (wasOutOfZone && nowInZoneBullish) {
            pair.crossTime = new Date(latestData.datetime).toLocaleString()
            pair.alert = `Bullish crossover above -20 on ${pair.crossTime}`
            console.log(`${pairName} entered bullish zone at ${pair.crossTime}`)
            triggerAlert(pairName, pair.alert)
        } else if (wasOutOfZone && nowInZoneBearish) {
            pair.crossTime = new Date(latestData.datetime).toLocaleString()
            pair.alert = `Bearish crossover below -80 on ${pair.crossTime}`
            console.log(`${pairName} entered bearish zone at ${pair.crossTime}`)
            triggerAlert(pairName, pair.alert)
        }
    }

    forexPairs.value = [...forexPairs.value]
}

const getCardColor = (pair) => {
    if (pair.currentWilliams === null && !pair.error) return 'white'
    if (pair.error) return 'grey lighten-4'
    // In Zone (bullish or bearish): green or red
    return (pair.currentWilliams >= -20 || pair.currentWilliams <= -80) ?
        (pair.currentWilliams >= -20 ? 'green lighten-5' : 'red lighten-5') :
        'grey lighten-2' // Out of zone (neutral)
}

const getIconColor = (pair) => {
    if (pair.currentWilliams === null && !pair.error) return 'grey'
    if (pair.error) return 'grey darken-1'
    return (pair.currentWilliams >= -20 || pair.currentWilliams <= -80) ?
        (pair.currentWilliams >= -20 ? 'green darken-1' : 'red darken-1') :
        'grey darken-2'
}

const getStatusColor = (pair) => {
    if (pair.currentWilliams === null && !pair.error) return 'grey'
    if (pair.error) return 'warning'
    if (pair.alert) return 'info'
    return (pair.currentWilliams >= -20 || pair.currentWilliams <= -80) ?
        (pair.currentWilliams >= -20 ? 'success' : 'error') :
        'grey'
}

const getStatusText = (pair) => {
    if (pair.error) return 'Error'
    if (pair.currentWilliams === null) return 'No Data'
    if (pair.alert) return 'Alert'
    return (pair.currentWilliams >= -20 || pair.currentWilliams <= -80) ?
        (pair.currentWilliams >= -20 ? 'Bullish' : 'Bearish') :
        'Neutral'
}

onMounted(() => {
    const endpoints = {
        'AUD/USD': 'https://edalertad.ertaccess.cc/data/1D',
        'EUR/USD': 'https://edalerteur.ertaccess.cc/data/1D',
        'NZD/USD': 'https://edalertnzd.ertaccess.cc/data/1D',
        'USD/CAD': 'https://edalertcad.ertaccess.cc/data/1D',
        'USD/CHF': 'https://edalertchf.ertaccess.cc/data/1D',
        'USD/JPY': 'https://edalertjpy.ertaccess.cc/data/1D'
    }

    const fetchAllData = async () => {
        const promises = Object.entries(endpoints).map(async ([pairName, url]) => {
            try {
                console.log(`Fetching data for ${pairName} from ${url}`)
                const response = await fetch(url)
                if (!response.ok) {
                    throw new Error(`HTTP error! status: ${response.status}`)
                }
                const data = await response.json()
                console.log(`${pairName} received ${data.length} records`)
                processData(data, pairName)
            } catch (error) {
                const pair = forexPairs.value.find(p => p.name === pairName)
                if (pair) {
                    pair.error = error.message || 'Fetch failed'
                    console.error(`Error fetching ${pairName}:`, error)
                }
            }
        })
        await Promise.all(promises)
    }

    requestNotificationPermission()
    fetchAllData()
    setInterval(fetchAllData, 3600000)
})
</script>

<style scoped>
.v-card {
    transition: transform 0.2s ease-in-out;
}

.v-card:hover {
    transform: translateY(-4px);
}
</style>