const API_KEY = "dfd4f9c43486df0fb4d0b4c7a99ca76f";
let weatherData = {};
let chart = null;
let thresholds = {
  maxTemp: null,
  unit: "celsius",
  consecutiveBreaches: 0,
  alertTriggered: false,
};

async function fetchWeatherData() {
  const city = document.getElementById("citySelect").value;
  const tempUnit = document.getElementById("tempUnit").value;

  showLoader(true);

  try {
    // Get coordinates
    const geoUrl = `https://api.openweathermap.org/geo/1.0/direct?q=${city}&limit=1&appid=${API_KEY}`;
    const geoResponse = await fetch(geoUrl);
    const geoData = await geoResponse.json();

    if (geoData.length === 0) {
      throw new Error("City not found");
    }

    const { lat, lon } = geoData[0];

    // Fetch current weather data
    const weatherUrl = `https://api.openweathermap.org/data/2.5/weather?lat=${lat}&lon=${lon}&appid=${API_KEY}`;
    const weatherResponse = await fetch(weatherUrl);
    const currentData = await weatherResponse.json();

    // Fetch forecast data
    const forecastUrl = `https://api.openweathermap.org/data/2.5/forecast?lat=${lat}&lon=${lon}&appid=${API_KEY}`;
    const forecastResponse = await fetch(forecastUrl);
    const forecastData = await forecastResponse.json();
    // console.log(forecastData);

    // Process temperatures from forecast data
    const temps = forecastData.list.map((item) => item.main.temp);
    const avgTemp = temps.reduce((sum, temp) => sum + temp, 0) / temps.length;
    const maxTemp = Math.max(...temps);
    const minTemp = Math.min(...temps);

    // Convert temperatures to selected unit
    const avgTempConverted = convertTemperature(avgTemp, tempUnit);
    const maxTempConverted = convertTemperature(maxTemp, tempUnit);
    const minTempConverted = convertTemperature(minTemp, tempUnit);

    // Determine dominant weather condition
    const conditionCounts = {};
    forecastData.list.forEach((item) => {
      const condition = item.weather[0].main;
      if (conditionCounts[condition]) {
        conditionCounts[condition]++;
      } else {
        conditionCounts[condition] = 1;
      }
    });
    const dominantCondition = Object.keys(conditionCounts).reduce((a, b) =>
      conditionCounts[a] > conditionCounts[b] ? a : b
    );

    // Store data
    weatherData[city] = {
      temp: convertTemperature(currentData.main.temp, tempUnit),
      feels_like: convertTemperature(currentData.main.feels_like, tempUnit),
      condition: currentData.weather[0].main,
      timestamp: currentData.dt,
      humidity: currentData.main.humidity,
      wind: currentData.wind.speed,
      unit: tempUnit,
      avgTemp: avgTempConverted,
      maxTemp: maxTempConverted,
      minTemp: minTempConverted,
      dominantCondition: dominantCondition,
    };
    // console.log(weatherData);
    updateUI();
    updateChart();
    checkThresholds();
  } catch (error) {
    showAlert(`Error fetching weather data: ${error.message}`);
  } finally {
    showLoader(false);
  }
}

function convertTemperature(kelvin, unit) {
  const celsius = kelvin - 273.15;
  return unit === "celsius" ? celsius : (celsius * 9) / 5 + 32;
}

function updateUI() {
  const grid = document.getElementById("weatherGrid");
  grid.innerHTML = "";

  for (const [city, data] of Object.entries(weatherData)) {
    const card = createWeatherCard(city, data);
    grid.appendChild(card);
  }
}

function createWeatherCard(city, data) {
  const div = document.createElement("div");
  div.className = "weather-card";

  const unit = data.unit === "celsius" ? "°C" : "°F";
  const cityName = city.split(",")[0];

  div.innerHTML = `
    <h3>${cityName}</h3>
    <div class="weather-info">
      <p>Temperature: ${data.temp.toFixed(1)}${unit}</p>
      <p>Feels Like: ${data.feels_like.toFixed(1)}${unit}</p>
      <p>Average Temp: ${data.avgTemp.toFixed(1)}${unit}</p>
      <p>Max Temp: ${data.maxTemp.toFixed(1)}${unit}</p>
      <p>Min Temp: ${data.minTemp.toFixed(1)}${unit}</p>
      <p>Dominant Condition: ${
        data.dominantCondition
      } (most frequent in forecast)</p>
      <p>Humidity: ${data.humidity} </p>
      <p>Wind Speed: ${data.wind} </p>
      <p>Condition: ${data.condition}</p>
      <p>Last Updated: ${new Date(
        data.timestamp * 1000
      ).toLocaleTimeString()}</p>
    </div>
  `;

  return div;
}

function updateChart() {
  const ctx = document.getElementById("weatherChart").getContext("2d");

  if (chart) {
    chart.destroy();
  }

  const labels = Object.keys(weatherData).map((city) => city.split(",")[0]);
  const temperatures = Object.values(weatherData).map((data) => data.temp);
  const avgTemps = Object.values(weatherData).map((data) => data.avgTemp);
  const maxTemps = Object.values(weatherData).map((data) => data.maxTemp);
  const minTemps = Object.values(weatherData).map((data) => data.minTemp);

  chart = new Chart(ctx, {
    type: "bar",
    data: {
      labels: labels,
      datasets: [
        {
          label: "Current Temperature",
          data: temperatures,
          backgroundColor: "rgba(54, 162, 235, 0.5)",
          borderColor: "rgba(54, 162, 235, 1)",
          borderWidth: 1,
        },
        {
          label: "Average Temperature",
          data: avgTemps,
          backgroundColor: "rgba(255, 206, 86, 0.5)",
          borderColor: "rgba(255, 206, 86, 1)",
          borderWidth: 1,
        },
        {
          label: "Max Temperature",
          data: maxTemps,
          backgroundColor: "rgba(255, 99, 132, 0.5)",
          borderColor: "rgba(255, 99, 132, 1)",
          borderWidth: 1,
        },
        {
          label: "Min Temperature",
          data: minTemps,
          backgroundColor: "rgba(75, 192, 192, 0.5)",
          borderColor: "rgba(75, 192, 192, 1)",
          borderWidth: 1,
        },
      ],
    },
    options: {
      responsive: true,
      scales: {
        y: {
          beginAtZero: false,
          title: {
            display: true,
            text: `Temperature (${
              weatherData[Object.keys(weatherData)[0]]?.unit === "celsius"
                ? "°C"
                : "°F"
            })`,
          },
        },
      },
    },
  });
}

function showLoader(show) {
  document.getElementById("loader").style.display = show ? "block" : "none";
}

function showAlert(message) {
  const alertContainer = document.getElementById("alertContainer");
  const alertDiv = document.createElement("div");
  alertDiv.className = "alert";
  alertDiv.textContent = message;
  alertContainer.appendChild(alertDiv);
  setTimeout(() => {
    alertDiv.remove();
  }, 5000);
}

function updateThresholds() {
  const maxTempInput = document.getElementById("maxTempThreshold").value;
  const maxTempUnit = document.getElementById("maxTempUnit").value;

  if (maxTempInput) {
    thresholds.maxTemp = parseFloat(maxTempInput);
    thresholds.unit = maxTempUnit;
    thresholds.consecutiveBreaches = 0;
    thresholds.alertTriggered = false;
    showAlert("Threshold updated successfully.");
  } else {
    showAlert("Please enter a valid maximum temperature threshold.");
  }
}

function checkThresholds() {
  const city = document.getElementById("citySelect").value;
  const data = weatherData[city];

  if (!data || !thresholds.maxTemp) {
    return;
  }

  let tempToCheck = data.temp;
  if (data.unit !== thresholds.unit) {
    // Convert temperature to threshold unit
    tempToCheck =
      data.unit === "celsius"
        ? (tempToCheck * 9) / 5 + 32
        : ((tempToCheck - 32) * 5) / 9;
  }

  if (tempToCheck > thresholds.maxTemp) {
    thresholds.consecutiveBreaches++;
    if (thresholds.consecutiveBreaches >= 2 && !thresholds.alertTriggered) {
      alert(
        `Temperature exceeded ${
          thresholds.maxTemp
        }°${thresholds.unit.toUpperCase()} for two consecutive updates!`
      );
      thresholds.alertTriggered = true;
    }
  } else {
    thresholds.consecutiveBreaches = 0;
    thresholds.alertTriggered = false;
  }
}

// Initial data fetch
fetchWeatherData();

// Add event listeners
document.getElementById("tempUnit").addEventListener("change", () => {
  // Convert existing data to new unit
  const newUnit = document.getElementById("tempUnit").value;
  for (const city in weatherData) {
    if (weatherData[city].unit !== newUnit) {
      const temp = weatherData[city].temp;
      const feels_like = weatherData[city].feels_like;
      const avgTemp = weatherData[city].avgTemp;
      const maxTemp = weatherData[city].maxTemp;
      const minTemp = weatherData[city].minTemp;

      if (newUnit === "fahrenheit") {
        weatherData[city].temp = (temp * 9) / 5 + 32;
        weatherData[city].feels_like = (feels_like * 9) / 5 + 32;
        weatherData[city].avgTemp = (avgTemp * 9) / 5 + 32;
        weatherData[city].maxTemp = (maxTemp * 9) / 5 + 32;
        weatherData[city].minTemp = (minTemp * 9) / 5 + 32;
      } else {
        weatherData[city].temp = ((temp - 32) * 5) / 9;
        weatherData[city].feels_like = ((feels_like - 32) * 5) / 9;
        weatherData[city].avgTemp = ((avgTemp - 32) * 5) / 9;
        weatherData[city].maxTemp = ((maxTemp - 32) * 5) / 9;
        weatherData[city].minTemp = ((minTemp - 32) * 5) / 9;
      }
      weatherData[city].unit = newUnit;
    }
  }
  updateUI();
  updateChart();
});

// Optional: set an interval to fetch data periodically
setInterval(fetchWeatherData, 60000); // Update every 60 seconds
