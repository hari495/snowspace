<div class="container">
	<!-- Snowflakes -->
	<div class="snowflakes" aria-hidden="true">
		{#each Array(20) as _, i}
			<div class="snowflake" style="left: {Math.random() * 100}%; animation-delay: {Math.random() * 10}s; animation-duration: {8 + Math.random() * 8}s;">
				❄
			</div>
		{/each}
	</div>

	<div class="auth-section">
		<h1 class="title">SnowSpace</h1>
		<p class="tagline">Tagline goes here</p>

		<img src="/icons/snowspace-logo-lm.svg" alt="SnowSpace Logo" class="logo" />

		<div class="divider">
			<span class="line"></span>
			<span class="divider-text">CONTINUE WITH</span>
			<span class="line"></span>
		</div>

		<div class="auth-buttons">
			<button class="auth-button" aria-label="Continue with Google">
				<img src="/icons/google-auth-icon.svg" alt="Google" />
			</button>
			<button class="auth-button" aria-label="Continue with Apple">
				<img src="/icons/apple-auth-icon.svg" alt="Apple" />
			</button>
			<button class="auth-button" aria-label="Continue with Phone">
				<img src="/icons/phone-auth-icon.svg" alt="Phone" />
			</button>
		</div>

		<a href="/home" class="skip-button">Skip</a>
	</div>
</div>

<style>
	.container {
		position: relative;
	}

	.snowflakes {
		position: absolute;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		pointer-events: none;
		z-index: 1;
		overflow: hidden;
	}

	.snowflake {
		position: absolute;
		top: -10%;
		color: #fff;
		font-size: 1.5em;
		opacity: 0.8;
		animation: fall linear infinite;
		text-shadow: 0 0 5px rgba(255, 255, 255, 0.5);
	}

	@keyframes fall {
		0% {
			top: -10%;
			transform: translateX(0px) rotate(0deg);
		}
		100% {
			top: 110%;
			transform: translateX(30px) rotate(360deg);
		}
	}

	.auth-section {
		position: relative;
		z-index: 2;
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		min-height: 100vh;
		gap: 0;
		text-align: center;
		padding-bottom: 20vh;
	}

	.title {
		font-family: var(--font-title);
		font-size: 48px;
		font-weight: normal;
		margin: 0;
		color: #4DB6ED;
		margin-bottom: 4px;
	}

	.tagline {
		font-family: var(--font-subtitle);
		font-style: italic;
		font-size: 18px;
		color: #666;
		margin: 0;
		margin-bottom: 16px;
	}

	.logo {
		width: 240px;
		height: 240px;
		margin: 0;
	}

	.divider {
		display: flex;
		align-items: center;
		width: 100%;
		max-width: 320px;
		gap: var(--spacing-sm);
		margin-top: var(--spacing-sm);
	}

	.line {
		flex: 1;
		height: 1px;
		background-color: var(--color-border);
	}

	.divider-text {
		font-family: var(--font-body);
		font-size: 11px;
		font-weight: 500;
		letter-spacing: 0.5px;
		color: #999;
	}

	.auth-buttons {
		display: flex;
		gap: var(--spacing-md);
		margin-top: var(--spacing-sm);
	}

	.auth-button {
		width: 64px;
		height: 64px;
		border-radius: 50%;
		border: 1.5px solid var(--color-border);
		background-color: #ffffff;
		display: flex;
		align-items: center;
		justify-content: center;
		cursor: pointer;
		transition: all 0.2s ease;
		padding: 0;
	}

	.auth-button:hover {
		transform: scale(1.05);
		border-color: #ccc;
		box-shadow: 0 4px 12px rgba(0, 0, 0, 0.08);
	}

	.auth-button:active {
		transform: scale(0.98);
	}

	.auth-button img {
		width: 32px;
		height: 32px;
		object-fit: contain;
	}

	.skip-button {
		margin-top: var(--spacing-lg);
		padding: 12px 48px;
		background-color: transparent;
		border: 1.5px solid var(--color-border);
		border-radius: 24px;
		font-family: var(--font-subtitle);
		font-style: italic;
		font-size: 16px;
		color: #666;
		text-decoration: none;
		cursor: pointer;
		transition: all 0.2s ease;
		display: inline-block;
	}

	.skip-button:hover {
		border-color: #999;
		color: #333;
		background-color: rgba(0, 0, 0, 0.02);
	}

	.skip-button:active {
		transform: scale(0.98);
	}
</style>
