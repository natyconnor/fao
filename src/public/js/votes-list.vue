<template>
	<ul class="votes-list">
		<li v-for="vote in Object.entries(votes)" :key="vote[0]">
			<span class="username" :style="{ color: color(vote[0]) }">
				{{ `${vote[0]} - ${vote[1]}` }}
			</span>
		</li>
	</ul>
</template>
<script>
import Store from './state';
export default {
	name: 'VotesList',
	props: {
		votes: {
			type: Object,
		},
	},
	methods: {
		color(username) {
			return Store.state.gameState.getUserColor(username);
		},
	},
};
</script>