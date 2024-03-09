<script setup>
import axios from "axios";
import { ref } from "vue";

let climate = ref("");
let dailyData = ref("");
let city = ref("");
let loading = ref(false);

function fetchClimate(lat, lon) {
  loading.value = true;
  axios
    .get(
      `https://api.openweathermap.org/data/2.5/forecast?lat=${lat}&lon=${lon}&appid=0eebd1fcf852d29ca0340c5c451d4c9a&units=metric`
    )
    .then((response) => {
      climate.value = response.data;
      dailyData.value = dailyTemps(response.data.list);
      console.log(dailyData.value);
      loading.value = false;
    })
    .catch((error) => {
      console.error(error);
    });
}

function dailyTemps(timeStamps) {
  const dailyData = [];

  timeStamps.forEach((timeStamp) => {
    const date = timeStamp.dt_txt.split(" ")[0]; // Extract date part

    // Check if the date already exists in the dailyData array
    const existingDateIndex = dailyData.findIndex((day) => day.date === date);

    if (existingDateIndex !== -1) {
      // If the date exists, push the timestamp to the existing array
      dailyData[existingDateIndex].timestamps.push(timeStamp);

      // Update min and max temperature if necessary
      const temperature = timeStamp.main.temp;
      dailyData[existingDateIndex].min = Math.min(
        dailyData[existingDateIndex].min,
        temperature
      );
      dailyData[existingDateIndex].max = Math.max(
        dailyData[existingDateIndex].max,
        temperature
      );
      dailyData[existingDateIndex].sum += temperature;
    } else {
      // If the date does not exist, create a new object and push it to the array
      dailyData.push({
        date: date,
        avg_temp: 0,
        min: Infinity,
        max: -Infinity,
        timestamps: [timeStamp],
        sum: timeStamp.main.temp,
      });
    }
  });

  // Calculate average temperature for each day
  dailyData.forEach((day) => {
    day.avg_temp = day.sum / day.timestamps.length;
  });

  return dailyData;
}

function extractHour(dateTimeString) {
  // Split the date time string into date and time parts
  const parts = dateTimeString.split(" ");

  // Extract the time part from the split array
  const time = parts[1];

  // Extract only the hour part from the time string
  const hour = time.split(":")[0];

  // Return the hour part
  if (hour >= 12) {
    return hour + ":00 pm";
  } else {
    return hour + ":00 am";
  }
}

function formatDate(dateString) {
  // Create a Date object from the input date string
  const date = new Date(dateString);

  // Get the month name from the Date object
  const monthName = date.toLocaleString("en-us", { month: "long" });

  // Get the day and year from the Date object
  const day = date.getDate();
  const year = date.getFullYear();

  // Return the formatted date string
  return monthName + " " + (day < 10 ? "0" + day : day) + " " + year;
}

function fetchCity(city) {
  axios
    .get(`https://search.reservamos.mx/api/v2/places?q=${city}`)
    .then((response) => {
      console.log(response.data[0]);
      const lat = response.data[0].lat;
      const long = response.data[0].long;
      fetchClimate(lat, long);
    })
    .catch((error) => {
      console.error(error);
    });
}
function getDayOfWeek(dateString) {
  // Create a Date object from the input date string
  const date = new Date(dateString);

  // Define an array of weekday names
  const daysOfWeek = [
    "Sunday",
    "Monday",
    "Tuesday",
    "Wednesday",
    "Thursday",
    "Friday",
    "Saturday",
  ];

  // Get the day of the week from the Date object
  const dayOfWeekIndex = date.getDay();

  // Return the corresponding weekday name
  return daysOfWeek[dayOfWeekIndex];
}
</script>

<template>
  <main>
    <div class="min-h-screen bg-slate-100 p-5 lg:p-10">
      <div v-if="!loading && climate" class="w-full px-10">
        <input
          v-model="city"
          class="rounded-l-full border border-slate-300 w-3/4 lg:w-1/2 px-5"
          type="text"
          name="city"
          id=""
          placeholder="Monterrey"
        />
        <button
          @click="fetchCity(city)"
          class="rounded-r-full bg-slate-700 text-white px-2 hover:bg-slate-800 border-slate-800"
        >
          Search
        </button>
      </div>
      <div
        class="mx-10 min-h-screen flex items-center justify-center"
        v-if="loading"
      >
        <div
          class="w-40 h-40 border-t rounded-full animate-spin border-black"
        ></div>
      </div>
      <div
        class="mx-10 min-h-screen flex items-center justify-center"
        v-if="!loading && !climate"
      >
        <div class="w-full h-40 text-center text-lg text-slate-600">
          <div
            class="flex flex-wrap items-center justify-center w-full lg:px-10"
          >
            <input
              v-model="city"
              class="rounded-lg lg:rounded-l-full border border-slate-300 lg:w-1/2 px-3 w-full"
              type="text"
              name="city"
              placeholder="Monterrey"
            />
            <button
              @click="fetchCity(city)"
              class="rounded-full mt-3 lg:mt-0 lg:rounded-l-none lg:rounded-r-full bg-slate-700 text-white px-2 hover:bg-slate-800 border-slate-800"
            >
              Search
            </button>
          </div>
          Search for a city you would love to know the weather
        </div>
      </div>
      <div v-if="climate && !loading" class="mx-3 lg:mx-10 mt-10">
        <h1 class="text-2xl lg:text-5xl font-semibold text-slate-700">
          {{ climate.city.name }}
        </h1>
      </div>
      <div
        v-if="climate && !loading"
        class="w-full grid grid-cols-2 lg:grid-cols-6 mt-5 lg:px-5"
      >
        <div
          class="bg-gradient-to-r from-slate-200 to-zinc-200 p-3 lg:p-5 rounded-lg font-semibold text-slate-700 m-3 lg:my-0 lg:mx-3"
          v-for="dayData in dailyData"
        >
          <p class="text-center mb-2">{{ getDayOfWeek(dayData.date) }}</p>
          <div class="flex items-center justify-center">
            <div class="flex space-x-5">
              <div class="flex flex-wrap text-right">
                <p class="w-full text-3xl text-sky-500">
                  {{ Math.ceil(dayData.min) }}°
                </p>
                <p class="w-full text-3xl font-light text-slate-700">min</p>
              </div>
              <div class="flex flex-wrap">
                <p class="w-full text-3xl text-red-500">
                  {{ Math.ceil(dayData.max) }}°
                </p>
                <p class="w-full text-3xl font-light text-slate-700">max</p>
              </div>
            </div>
          </div>
        </div>
      </div>
      <div
        v-if="climate && !loading"
        v-for="(dayData, date) in dailyData"
        :key="date"
        class="bg-gradient-to-r from-slate-100 to-zinc-100 rounded-lg lg:p-10 m-3 lg:m-10 border border-slate-200 shadow-lg"
      >
        <div class="flex space-x-2 justify-center m-5">
          <p class="text-2xl font-semibold text-slate-700">
            {{ formatDate(dayData.date) }}
          </p>
        </div>

        <div class="w-full grid grid-cols-2">
          <div
            class="flex flex-wrap items-center justify-center text-3xl lg:text-[100px] font-semibold text-slate-700"
          >
            {{ Math.ceil(dayData.avg_temp) }}°C
            <span class="text-sm ml-5">Avg trought the day.</span>
          </div>
          <div class="flex items-center justify-center">
            <div class="flex space-x-5">
              <div class="flex flex-wrap text-right">
                <p class="w-full text-3xl text-sky-500">
                  {{ Math.ceil(dayData.min) }}°
                </p>
                <p class="w-full text-3xl font-light text-slate-700">min</p>
              </div>
              <div class="flex flex-wrap">
                <p class="w-full text-3xl text-red-500">
                  {{ Math.ceil(dayData.max) }}°
                </p>
                <p class="w-full text-3xl font-light text-slate-700">max</p>
              </div>
            </div>
          </div>
        </div>
        <div class="grid grid-cols-2 lg:grid-cols-8 border-t pt-10">
          <div
            class="bg-white m-1 shadow-md flex flex-wrap items-center justify-center border rounded-lg p-2"
            v-for="(timeStamp, index) in dayData.timestamps"
            :key="index"
          >
            <p class="text-sm font-light text-slate-500">
              {{ extractHour(timeStamp.dt_txt) }}
            </p>
            <p
              class="text-center w-full text-4xl font-bold text-slate-700 border-b pb-2 mb-2"
            >
              {{ Math.ceil(timeStamp.main.temp) }}C°
            </p>
          </div>
        </div>
      </div>
    </div>
  </main>
</template>
