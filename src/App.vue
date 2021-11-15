<template>
  <div id="app" class="app-container">
    
    <!-- Login Form -->
    <div v-if="!user.username && !newUser" class="container-100">
			<div class="wrap-login100">
				<div class="login100-pic">
					<img src="./assets/img-01.png" alt="Login Image">
				</div>

				<form v-on:submit.prevent autocomplete="off" class="login100-form">
					<span class="login100-form-title">
						Welcome to TempTracker
					</span>

					<div class="wrap-input100">
            <input v-model='form.username' placeholder="Username" type="username" class='input100' autocomplete="off"/> 
						<span class="focus-input100"></span>
					</div>
					
					<div class="container-100-form-btn">
						<button @click='login' class="login100-form-btn">
							Login
						</button>
					</div>

					<div class="text-center p-t-136">
						<a @click="newUser=true" v-on:click.prevent class="txt2">
							Create new Username
						</a>
					</div>
				</form>
			</div>
		</div>



    <!-- Create New User Form -->
    <div v-if="newUser" class="container-100">
			<div class="wrap-login100">
				<div class="login100-pic">
					<img src="./assets/img-01.png" alt="Login Image">
				</div>

				<form v-on:submit.prevent autocomplete="off" class="login100-form">
					<span class="login100-form-title">
						Create a new User
					</span>

					<div class="wrap-input100">
            <input v-model='form.username' placeholder="Username" type="username" class='input100' autocomplete="off"/> 
						<span class="focus-input100"></span>
					</div>
					
					<div class="container-100-form-btn">
						<button @click='createUser' class="register100-form-btn">
							Register and Login
						</button>
					</div>

					<div class="text-center p-t-136">
						<a @click="newUser=false" v-on:click.prevent class="txt2" href="#">
							Return to Login
						</a>
					</div>
				</form>
			</div>
		</div>



    <!-- Weather App -->
    <div v-if="user.username" class="container mb-4">
      <div class="welcome bg-white rounded-3 mb-2 p-3 mt-4"  style="display: inline-block">
        <span class="login100-form-title">
          Welcome, {{formattedUsername}}
        </span>
        <div class="d-inline-flex">
          <form v-on:submit.prevent autocomplete="off" class="login100-form">
            <div>
              <label class="text-start">Add new location: </label>
              <input v-model='form.zipcode' placeholder="Zipcode" type="zipcode" class='input100' autocomplete="off"/>
            </div>
              <button @click='updateUser' class='btn btn-success m-2'>Add</button>
              <!-- <button @click='getWeatherData' class='button'>Refresh</button> -->
          </form>
        </div>
      </div>
      <h2 v-if="loading" class="loading text-white">Loading...</h2>
      <div v-if="!loading" class="row row-cols-1 row-cols-md-2 g-4 mt-1 text-center justify-content-center">
        <div  v-for="(location, i) in user.locations" :key="location" class="col">
          <div class="card" style="display: block">
            <img v-bind:src='weatherData[i].icon' class="card-img-top text-center" alt="..." style="max-height:25%; max-width:25%">
            <div class="card-body">
              <p class="card-text">Zipcode: {{ location }}</p>
              <h5 class="card-title">{{weatherData[i].name}}<br/>{{weatherData[i].main.temp}}&#176; F</h5>
              <button @click='deleteLocation(location)' class='btn btn-danger m-2'>Delete</button>
            </div>
          </div>
        </div>
      </div>
    </div>



  </div>
</template>

<script>
import { API, graphqlOperation } from 'aws-amplify'
import * as queries from './graphql/queries'
import { createUser, updateUser } from './graphql/mutations'


export default {
  name: 'App',
  data() {
    return {
      user: { 
        // username:"dimitri", 
        // locations: [40511, 15120, 30549] 
        },
      weatherData: [],
      form: { },
      loading: false,
      newUser: false
    }
  },
  computed: {
    formattedUsername() {
      let username = this.user.username;
      return username.charAt(0).toUpperCase() + username.slice(1);
    }
  },
  methods: {
    async login() {
      this.loading = true;
      if (!this.form.username) return;
      try {
        const response = await API.graphql(graphqlOperation(queries.getUser, {username: this.form.username.toLowerCase()}));
        if (response.data.getUser.username) {
          this.user = response.data.getUser;
        } else {
          alert('That Username does not exist.');
        }
      }
      catch (error) {
        console.log('Error logging in...', error);
        alert('That Username does not exist.');
      }
      finally {
        this.form.username = "";
        if(this.user.locations.length > 0) {
          this.getWeatherData();
        }
        else {
          this.loading = false;
        }
        this.newUser = false;
      }
    },

    async createUser() {
      this.loading = true;
      if (!this.form.username) return;
      const newUser = { username: this.form.username.toLowerCase() };
      try {
        const response = await API.graphql(graphqlOperation(createUser, { input: newUser }))
        if (response.data) this.login();
      }
      catch (error) {
        console.log('Error creating new user. Error:', error);
        alert('This username already exists');
        this.loading = false;
      }
      finally {
        this.loading = false;
      }
    },

    async updateUser() {
      this.loading = true;
      if (!this.form.zipcode) return;
      let newZipcode = parseInt(this.form.zipcode);

      if(this.user.locations) {
        if(this.user.locations.includes(newZipcode)){
          alert("This zipcode is already being tracked.");
          this.form.zipcode = '';
          this.loading = false;
          return;
        }
      }

      let newUser = {username: this.user.username, locations:this.user.locations}
      if (newUser.locations) {
        newUser.locations.push(newZipcode);
      } else {
        newUser.locations = [newZipcode]
      }
      
      try {
        this.user = newUser;
        const response = await API.graphql(graphqlOperation(updateUser, { input: newUser }))
        if (response.data) {
          this.getWeatherData();
        }
      }
      catch (error) {
        console.log('Error adding new zipcode. Error:', error);
        alert('Error while adding zipcode: ', newZipcode);
      }
      finally {
        this.form.zipcode = "";
      }
    },

    async deleteLocation(location) {
      this.loading = true;
      let index = this.user.locations.indexOf(location);
      let newLocations = this.user.locations;
      newLocations.splice(index, 1);
      let newUser = { username: this.user.username, locations: newLocations }
      try {
        await API.graphql(graphqlOperation(updateUser, { input: newUser }))
      }
      catch (error) {
        console.log('Error deleting zipcode. Error:', error);
        alert('Error while deleting zipcode: ', location);
      }
      finally {
        this.user.locations = newLocations
        console.log('end of deleteLocation', newLocations.length)
        if (newLocations.length > 0) this.getWeatherData();
        else this.loading = false
      }
    },
    
    async getWeatherData() {
      let apiKey = "f028d85563e1ac4ff659b7cd4064a0c1";
      let locations = this.user.locations;
      this.weatherData = [];
      for (let i=0; i<locations.length; i++) {
        try {
          let url = `http://api.openweathermap.org/data/2.5/weather?zip=${locations[i]}&units=imperial&appid=${apiKey}`
          const response = await this.$http.get(url);
          // JSON responses are automatically parsed.
          if(response){
            this.weatherData[i] = response.data;
            this.weatherData[i].id = i;
            this.weatherData[i].icon = 
            `https://s3-us-west-2.amazonaws.com/s.cdpn.io/162656/${this.weatherData[i].weather[0]["icon"]}.svg`
          } 
        } 
        catch (error) {
          console.log(error);
          this.weatherData[i] = { name: "Unknown", icon: `https://s3-us-west-2.amazonaws.com/s.cdpn.io/162656/50d.svg`, id: i };
        } 
        finally {
          console.log(this.weatherData.length, locations.length)
          if (this.weatherData.length == locations.length) this.loading = false;
        }
      }      
    }
  }, 
  components: {
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;  
}
.app-container {
    background: #092756;
    background: -moz-radial-gradient(0% 100%, ellipse cover, rgba(104,128,138,.4) 10%,rgba(138,114,76,0) 40%),-moz-linear-gradient(top,  rgba(57,173,219,.25) 0%, rgba(42,60,87,.4) 100%), -moz-linear-gradient(-45deg,  #670d10 0%, #092756 100%);
    background: -webkit-radial-gradient(0% 100%, ellipse cover, rgba(104,128,138,.4) 10%,rgba(138,114,76,0) 40%), -webkit-linear-gradient(top,  rgba(57,173,219,.25) 0%,rgba(42,60,87,.4) 100%), -webkit-linear-gradient(-45deg,  #670d10 0%,#092756 100%);
    background: -o-radial-gradient(0% 100%, ellipse cover, rgba(104,128,138,.4) 10%,rgba(138,114,76,0) 40%), -o-linear-gradient(top,  rgba(57,173,219,.25) 0%,rgba(42,60,87,.4) 100%), -o-linear-gradient(-45deg,  #670d10 0%,#092756 100%);
    background: -ms-radial-gradient(0% 100%, ellipse cover, rgba(104,128,138,.4) 10%,rgba(138,114,76,0) 40%), -ms-linear-gradient(top,  rgba(57,173,219,.25) 0%,rgba(42,60,87,.4) 100%), -ms-linear-gradient(-45deg,  #670d10 0%,#092756 100%);
    background: -webkit-radial-gradient(0% 100%, ellipse cover, rgba(104,128,138,.4) 10%,rgba(138,114,76,0) 40%), linear-gradient(to bottom,  rgba(57,173,219,.25) 0%,rgba(42,60,87,.4) 100%), linear-gradient(135deg,  #670d10 0%,#092756 100%);
    min-height: 100vh;
  }
</style>
