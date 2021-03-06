<template>
	<div id="in-game" class="view">
		<div class="view-container">
			<room-info
				@close="hideRoomInfo"
				:users="gameState.users"
				:room-code="gameState.roomCode"
				v-show="roomInfoDialogVisible"
			></room-info>
			<confirmation
				id="confirm-skip-dialog"
				v-show="skipRoundConfirmationDialogVisible"
				@close="hideSkipRoundConfirmationDialog"
				@confirm="skip"
			>
				<h2>Skip this Round?</h2>
				<div class="normal-text">
					<p>
						This will end the current round.
					</p>
				</div>
			</confirmation>
			<confirmation
				id="confirm-setup-dialog"
				v-show="setupConfirmationDialogVisible"
				@close="hideSetupConfirmationDialog"
				@confirm="setup"
			>
				<h2>Exit to Setup?</h2>
				<div class="normal-text">
					<p>
						Returning to setup will let you add/remove players. This will end the
						current round.
					</p>
				</div>
			</confirmation>
			<div class="stripe">
				<div id="game-info" class="stripe-content canvas-aligned">
					<h1 class="username">{{ username }}</h1>
					<h1 class="prompt" v-show="promptVisible">{{ promptText }}</h1>
					<h2 class="current-turn" :style="{ color: userColor }">{{ whoseTurnText }}</h2>
					<h3 class="fake-guessing-info" v-show="isFakerGuessing && !userIsFaker">
						Fake Artist is guessing...
					</h3>
					<div class="faker-guess-form" v-if="isFakerGuessing && userIsFaker">
						<input
							type="text"
							class="faker-guess-input"
							placeholder="Guess the word"
							required
							autocomplete="off"
							v-model="fakerGuess"
						/>
						<button
							class="btn primary vote"
							@click="submitGuess"
						>
							Guess
						</button>
					</div>
					<h3 class="end-game-info" v-show="roundOver">
						{{ gameOverText() }}
					</h3>
				</div>
			</div>
			<div class="stripe flex-center">
				<div id="drawing-pad" class="stripe-content">
					<connection-overlay :gameConnection="gameConnection"></connection-overlay>
					<canvas
						id="new-paint"
						touch-action="none"
						@pointerdown="pdown"
						@pointermove="pmove"
						@pointerup="endStroke"
						@pointerout="endStroke"
					></canvas>
					<canvas id="old-paint"></canvas>
				</div>
			</div>
			<div id="drawing-actions" class="stripe flex-center">
				<div class="stripe-content flex-center canvas-aligned">
					<div id="drawing-actions-right" class="fill-space"></div>
					<div id="drawing-actions-center">
						<button
							class="btn primary big"
							@click="newRound"
							v-show="roundOver"
							:disabled="!roundOver"
						>
							New Round
						</button>
						<button
							class="btn primary submit-drawing"
							@click="submit"
							v-show="isDrawing"
							:disabled="!actionsEnabled"
						>
							Submit
						</button>
						<button
							class="btn secondary undo-drawing"
							@click="undo"
							v-show="isDrawing"
							:disabled="!actionsEnabled"
						>
							Undo
						</button>
						<select
							class="select"
							v-model="votedPlayer"
							v-if="isVoting"
						>
							<option disabled value="">Pick your vote</option>
							<option
								v-for="user in otherUsers"
								:key="'0' + user.name"
								:value="user.name"
							>
								{{ user.name }}
							</option>
						</select>
						<button
							class="btn primary vote"
							@click="vote"
							v-show="isVoting"
							:disabled="!isVoting || !votedPlayer || submittedVote"
						>
							Vote
						</button>
					</div>
					<div id="drawing-actions-left" class="fill-space">
						<game-menu :items="menuItems"></game-menu>
					</div>
				</div>
			</div>
		</div>
		<div
			id="side-player-statuses"
			v-if="playerStatusesListMaxWidth > 0"
			:style="{
				maxWidth: `${playerStatusesListMaxWidth}px`,
			}"
		>
			<h1>Players</h1>
			<PlayerStatusesList :users="gameState.users" />
		</div>
		<div
			id="side-votes"
			v-show="isDoneDrawing"
			v-if="playerStatusesListMaxWidth > 0"
			:style="{
				maxWidth: `${playerStatusesListMaxWidth}px`,
			}"
		>
			<h1>Votes</h1>
			<div v-show="isVoting">
				{{ waitingForVotesText }}
			</div>
			<VotesList v-show="isFakerGuessing || roundOver" :votes="gameState.votes" />
		</div>
	</div>
</template>

<script>
import Store from './state';
import VIEW from './view';
import Layer from './layer';
import RelativePoint from '../../common/relative-point';
import GAME_PHASE from '../../common/game-phase';
import CONNECTION_STATE from './connection-state';
import ConnectionOverlay from './connection-overlay';
import GameMenu from './game-menu';
import RoomInfo from './room-info';
import Confirmation from './confirmation';
import drawingPad from './drawing-pad';
import PlayerStatusesList from './player-statuses-list';
import VotesList from './votes-list';

const CanvasState = {
	EMPTY: 'EMPTY',
	PAINT: 'PAINT',
	PREVIEW: 'PREVIEW',
	SPECTATE: 'SPECTATE',
};

const strokeTracker = {
	points: [],
	maxCount: 5000,
	strokeLength: 0,
	addPoint: function(p) {
		if (this.points.length < this.maxCount) {
			this.points.push(p);
		}
		return this.points;
	},
	lastPoint: function() {
		let prevPt = this.points[this.points.length - 1];
		return prevPt;
	},
	reset: function() {
		this.points = [];
		this.strokeLength = 0;
	},
	validateStrokeDistance: function() {
		// TODO validate by relativeLength?
		if (this.points.length < 2) {
			return false;
		}
		// console.log(this.points.length);
		const minLength = 0.02;
		let dist = 0;
		for (let i = 1; i < this.points.length; i++) {
			let prevPt = this.points[i - 1];
			let curPt = this.points[i];
			let a = prevPt.x - curPt.x;
			let b = prevPt.y - curPt.y;
			dist += Math.sqrt(a * a + b * b);
			// console.log(dist);
			if (dist > minLength) {
				return true;
			}
		}
		return false;
	},
	hasPoints: function() {
		return this.points.length > 0;
	},
};

const SIDE_PLAYER_STATUSES_LIST_MIN_WIDTH = 120;

export default {
	name: 'GameView',
	components: {
		ConnectionOverlay,
		GameMenu,
		RoomInfo,
		Confirmation,
		PlayerStatusesList,
		VotesList,
	},
	props: {
		gameConnection: {
			type: String,
			required: true,
		},
		gameState: {
			type: Object,
			required: true,
		},
	},
	data() {
		return {
			canvasState: CanvasState.SPECTATE,
			stroke: strokeTracker,
			drawingPad: drawingPad,
			promptVisible: true,
			skipRoundConfirmationDialogVisible: false,
			setupConfirmationDialogVisible: false,
			menuItems: [
				{
					text: 'Toggle prompt visibility',
					action: this.togglePrompt,
				},
				{
					text: 'View room & player info',
					action: this.showRoomInfo,
				},
				{
					text: 'break1',
					hr: true,
				},
				{
					// 	text: 'Rules',
					// 	action: this.rules,
					// }, {
					// 	text: 'break2',
					// 	hr: true,
					// }, {
					text: 'Skip this round',
					action: this.showSkipRoundConfirmationDialog,
				},
				{
					text: 'Exit to setup',
					action: this.showSetupConfirmationDialog,
				},
			],
			roomInfoDialogVisible: false,
			playerStatusesListMaxWidth: 0,
			votedPlayer: '',
			submittedVote: false,
			fakerGuess: '',
		};
	},
	computed: {
		promptText() {
			return `${this.gameState.keyword} (${this.gameState.hint})`;
		},
		username() {
			return Store.getMyUsername();
		},
		userIsFaker() {
			return this.gameState.fakerName === Store.getMyUsername();
		},
		whoseTurnText() {
			switch (this.gameState.phase) {
				case GAME_PHASE.VOTE:
					return 'Time to vote!';
				case GAME_PHASE.FAKER_GUESS:
					if (this.userIsFaker) {
						return '🙀 You were caught! What do you think the word is? 🤔';
					}
					return '🕵️‍♂️ Great job, you caught the Fake Artist, ' + this.gameState.fakerName
						+ ', but they get to try and guess the word...';
				case GAME_PHASE.END:
					return 'Game Over!';
				default:
					return Store.myTurn()
						? 'Your Turn!'
						: `${this.gameState.whoseTurn}'s turn`;
			}
		},
		waitingForVotesText() {
			const numVotesLeft = this.gameState.users.length - Store.getNumVotes(this.gameState.votes);
			return 'Waiting for ' + numVotesLeft + " players to vote...";
		},
		userColor() {
			return this.gameState.getUserColor(this.gameState.whoseTurn);
		},
		isDrawing() {
			return this.gameState.phase === GAME_PHASE.PLAY;
		},
		isVoting() {
			return this.gameState.phase === GAME_PHASE.VOTE;
		},
		isDoneDrawing() {
			// any phase after drawing
			return this.gameState.phase === GAME_PHASE.VOTE || this.gameState.phase === GAME_PHASE.FAKER_GUESS || this.gameState.phase === GAME_PHASE.END;
		},
		isFakerGuessing() {
			return this.gameState.phase === GAME_PHASE.FAKER_GUESS;
		},
		roundOver() {
			return this.gameState.phase === GAME_PHASE.END;
		},
		actionsEnabled() {
			return (
				this.canvasState === 'PREVIEW' && this.gameConnection === CONNECTION_STATE.CONNECT
			);
		},
		roundAndTurn() {
			return this.gameState.round + '-' + this.gameState.turn;
		},
		otherUsers() {
			return this.gameState.users.filter(user => user.name !== Store.getMyUsername());
		}
	},
	watch: {
		roundAndTurn() {
			this.reset();
		},
		['gameState.round']() {
			this.promptVisible = true;
		},
	},
	methods: {
		gameOverText() {
			if (this.gameState.fakerCaught) {
				if (this.userIsFaker) {
					if (this.gameState.fakerGuessedCorrectly) {
						return '🧐 Nice you figured out the word!'
					}
					return '😭 Nope, the word was ' + this.gameState.keyword;
				}
				if (this.gameState.fakerGuessedCorrectly) {
					return '😖 Darn you caught them, but they guessed the word correctly!'
				}
				return '🕵️‍♂️ Great job, you caught the Fake Artist, ' + this.gameState.fakerName + ', and they didn\'t figure out the word!';
			}
			if (this.userIsFaker) {
				return '😎 Nice job! Not enough people found you out...';
			}
			return '🤦‍♂️ Oh no, the votes were wrong! The Fake Artist was ' + this.gameState.fakerName;
		},
		reset() {
			if (this.gameState.turn === 1) {
				drawingPad.clearLayer(Layer.BOTTOM);
			}

			drawingPad.clearLayer(Layer.TOP);
			this.stroke.reset();
			// TODO draw only the strokes that haven't been drawn yet (keeping connection loss in mind)
			for (let stroke of this.gameState.strokes) {
				drawingPad.drawStroke(
					Layer.BOTTOM,
					stroke.points,
					this.gameState.getUserColor(stroke.username)
				);
			}

			if (Store.myTurn()) {
				this.canvasState = CanvasState.EMPTY;
			} else {
				this.canvasState = CanvasState.SPECTATE;
			}
			this.votedPlayer = '';
			this.submittedVote = false;
			this.fakerGuess = '';
		},
		undo() {
			this.stroke.reset();
			drawingPad.clearLayer(Layer.TOP);
			this.canvasState = CanvasState.EMPTY;
		},
		submit() {
			if (Store.myTurn() && this.stroke.hasPoints()) {
				Store.submitStroke(this.stroke.points);

				this.stroke.reset();
				this.canvasState = CanvasState.SPECTATE;
			}
		},
		vote() {
			Store.submitVote(this.votedPlayer);
			this.submittedVote = true;
		},
		submitGuess() {
			Store.submitGuess(this.fakerGuess);
		},
		newRound() {
			Store.submitStartGame();
		},
		pdown(e) {
			if (this.canvasState === CanvasState.EMPTY && Store.myTurn()) {
				this.canvasState = CanvasState.PAINT;
				let newPt = drawingPad.getRelativePointFromPointerEvent(e);
				strokeTracker.addPoint(newPt);
			}
		},
		pmove(e) {
			if (this.canvasState === CanvasState.PAINT && Store.myTurn()) {
				let div = document.getElementById('new-paint');
				let lastPt = strokeTracker.lastPoint();
				let newPt = drawingPad.getRelativePointFromPointerEvent(e);
				if (!lastPt.matches(newPt)) {
					strokeTracker.addPoint(newPt);
					drawingPad.drawStroke(Layer.TOP, strokeTracker.points, 'black');
				}
			}
		},
		endStroke(e) {
			if (this.canvasState === CanvasState.PAINT && Store.myTurn()) {
				if (strokeTracker.validateStrokeDistance()) {
					this.canvasState = CanvasState.PREVIEW;
					let lastPt = strokeTracker.lastPoint();
					let newPt = drawingPad.getRelativePointFromPointerEvent(e);
					if (!lastPt.matches(newPt)) {
						strokeTracker.addPoint(newPt);
						drawingPad.drawStroke(Layer.TOP, strokeTracker.points, 'black');
					}
				} else {
					drawingPad.clearLayer(Layer.TOP);
					this.canvasState = CanvasState.EMPTY;
					strokeTracker.reset();
				}
			}
		},
		onWindowResize() {
			this.resizeDrawingPad();
			this.resizePlayerStatusesList();
		},
		resizeDrawingPad() {
			drawingPad.adjustSize();
			drawingPad.clearLayer(Layer.TOP);
			drawingPad.drawStroke(Layer.TOP, strokeTracker.points, 'black');
			drawingPad.clearLayer(Layer.BOTTOM);
			for (let stroke of this.gameState.strokes) {
				drawingPad.drawStroke(
					Layer.BOTTOM,
					stroke.points,
					this.gameState.getUserColor(stroke.username)
				);
			}
		},
		resizePlayerStatusesList() {
			const availableWidth = window.innerWidth / 2 - drawingPad.canvasWidth / 2;
			if (availableWidth >= SIDE_PLAYER_STATUSES_LIST_MIN_WIDTH) {
				this.playerStatusesListMaxWidth = Math.floor(availableWidth);
			} else {
				this.playerStatusesListMaxWidth = 0;
			}
		},
		togglePrompt() {
			this.promptVisible = !this.promptVisible;
		},
		showSkipRoundConfirmationDialog() {
			this.skipRoundConfirmationDialogVisible = true;
		},
		showSetupConfirmationDialog() {
			this.setupConfirmationDialogVisible = true;
		},
		hideSkipRoundConfirmationDialog() {
			this.skipRoundConfirmationDialogVisible = false;
		},
		hideSetupConfirmationDialog() {
			this.setupConfirmationDialogVisible = false;
		},
		skip() {
			Store.submitSkipRound();
			this.hideSkipRoundConfirmationDialog();
		},
		setup() {
			Store.submitReturnToSetup();
			this.hideSetupConfirmationDialog();
		},
		showRoomInfo() {
			this.roomInfoDialogVisible = true;
		},
		hideRoomInfo() {
			this.roomInfoDialogVisible = false;
		},
		rules() {
			Store.setView(VIEW.RULES);
		},
	},
	mounted() {
		this.$nextTick(function() {
			drawingPad.init();
			drawingPad.adjustSize();
			this.resizePlayerStatusesList();
			this.reset();
		});
		window.addEventListener('resize', this.onWindowResize);
	},
	beforeDestroy() {
		window.removeEventListener('resize', this.onWindowResize);
	},
};
</script>
