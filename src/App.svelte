<script>
  const PROFILE_KEY = 'maintainer.profile.v1';
  const ENTRIES_KEY = 'maintainer.entries.v1';
  const LB_PER_KG = 2.2046226218;
  const CM_PER_INCH = 2.54;

  const activityLevels = [
    { value: 'sedentary', label: 'Sedentary', factor: 1.2 },
    { value: 'light', label: 'Lightly active', factor: 1.375 },
    { value: 'moderate', label: 'Moderately active', factor: 1.55 },
    { value: 'very', label: 'Very active', factor: 1.725 },
    { value: 'athlete', label: 'Athlete', factor: 1.9 }
  ];

  const initialProfile = loadStored(PROFILE_KEY, null);

  let screen = $state('record');
  let profile = $state(initialProfile);
  let entries = $state(loadStored(ENTRIES_KEY, []));
  let profileForm = $state(profileToForm(initialProfile));
  let weightInput = $state('');
  let setupError = $state('');
  let weightError = $state('');
  let resetConfirming = $state(false);

  let sortedEntries = $derived(
    [...entries].sort((a, b) => a.date.localeCompare(b.date))
  );
  let analysis = $derived(calculateAnalysis(sortedEntries, profile));
  let tone = $derived(getTone(analysis?.calorieDifference ?? 0));
  let todayEntry = $derived(
    sortedEntries.find((entry) => entry.date === getTodayKey())
  );

  function loadStored(key, fallback) {
    if (typeof localStorage === 'undefined') return fallback;

    try {
      const raw = localStorage.getItem(key);
      return raw ? JSON.parse(raw) : fallback;
    } catch {
      return fallback;
    }
  }

  function saveStored(key, value) {
    localStorage.setItem(key, JSON.stringify(value));
  }

  function profileToForm(source) {
    if (!source) {
      return {
        units: 'imperial',
        birthYear: '',
        gender: 'male',
        heightFeet: '',
        heightInches: '',
        heightCm: '',
        activityLevel: 'moderate'
      };
    }

    const totalInches = source.heightCm / CM_PER_INCH;
    return {
      units: source.units,
      birthYear: String(source.birthYear),
      gender: source.gender,
      heightFeet: String(Math.floor(totalInches / 12)),
      heightInches: String(Math.round(totalInches % 12)),
      heightCm: String(Math.round(source.heightCm)),
      activityLevel: source.activityLevel
    };
  }

  function saveProfile(event) {
    event.preventDefault();
    setupError = '';

    const birthYear = Number(profileForm.birthYear);
    const currentYear = new Date().getFullYear();
    const age = currentYear - birthYear;
    let heightCm = 0;

    if (profileForm.units === 'metric') {
      heightCm = Number(profileForm.heightCm);
    } else {
      heightCm =
        (Number(profileForm.heightFeet) * 12 + Number(profileForm.heightInches || 0)) *
        CM_PER_INCH;
    }

    if (!Number.isInteger(birthYear) || age < 13 || age > 110) {
      setupError = 'Enter a birth year for an age between 13 and 110.';
      return;
    }

    if (!Number.isFinite(heightCm) || heightCm < 100 || heightCm > 250) {
      setupError = 'Enter a valid height.';
      return;
    }

    const nextProfile = {
      units: profileForm.units,
      birthYear,
      gender: profileForm.gender,
      heightCm,
      activityLevel: profileForm.activityLevel
    };

    profile = nextProfile;
    profileForm = profileToForm(nextProfile);
    saveStored(PROFILE_KEY, nextProfile);
    screen = 'record';
  }

  function saveWeight(event) {
    event.preventDefault();
    weightError = '';

    const rawWeight = Number(weightInput);
    if (!Number.isFinite(rawWeight) || rawWeight <= 0) {
      weightError = 'Enter a valid bodyweight.';
      return;
    }

    const weightLb = profile.units === 'metric' ? rawWeight * LB_PER_KG : rawWeight;
    if (weightLb < 60 || weightLb > 900) {
      weightError = 'Enter a bodyweight in the expected range.';
      return;
    }

    const date = getTodayKey();
    const withoutToday = sortedEntries.filter((entry) => entry.date !== date);
    entries = [...withoutToday, { date, weightLb }].sort((a, b) =>
      a.date.localeCompare(b.date)
    );
    saveStored(ENTRIES_KEY, entries);
    weightInput = '';
  }

  function confirmReset() {
    resetConfirming = true;
  }

  function cancelReset() {
    resetConfirming = false;
  }

  function resetApp() {
    localStorage.removeItem(PROFILE_KEY);
    localStorage.removeItem(ENTRIES_KEY);
    profile = null;
    entries = [];
    profileForm = profileToForm(null);
    weightInput = '';
    screen = 'record';
    resetConfirming = false;
  }

  function getTodayKey() {
    const now = new Date();
    const year = now.getFullYear();
    const month = String(now.getMonth() + 1).padStart(2, '0');
    const day = String(now.getDate()).padStart(2, '0');
    return `${year}-${month}-${day}`;
  }

  function calculateAnalysis(sourceEntries, sourceProfile) {
    if (!sourceProfile) return null;

    const total = sourceEntries.length;
    if (total < 4) {
      return {
        ready: false,
        daysNeeded: 4 - total,
        total
      };
    }

    const sampleDays = total < 8 ? total - 1 : 7;
    const baselineEntries = total < 8 ? sourceEntries.slice(0, -1) : sourceEntries.slice(0, 7);
    const rollingEntries = sourceEntries.slice(-sampleDays);
    const baselineAverageLb = average(baselineEntries.map((entry) => entry.weightLb));
    const rollingAverageLb = average(rollingEntries.map((entry) => entry.weightLb));
    const differenceLb = rollingAverageLb - baselineAverageLb;
    const calorieDifference = Math.max(0, differenceLb * 3500);
    const tdee = calculateTdee(sourceProfile);

    return {
      ready: true,
      total,
      sampleDays,
      baselineAverageLb,
      rollingAverageLb,
      differenceLb,
      calorieDifference,
      tdee,
      recommendedCalories: Math.max(0, tdee - calorieDifference)
    };
  }

  function average(values) {
    return values.reduce((sum, value) => sum + value, 0) / values.length;
  }

  function calculateTdee(sourceProfile) {
    const latestWeight = sortedEntries.at(-1)?.weightLb ?? 170;
    const weightKg = latestWeight / LB_PER_KG;
    const age = new Date().getFullYear() - sourceProfile.birthYear;
    const sexAdjustment = sourceProfile.gender === 'male' ? 5 : -161;
    const bmr = 10 * weightKg + 6.25 * sourceProfile.heightCm - 5 * age + sexAdjustment;
    const activity = activityLevels.find((level) => level.value === sourceProfile.activityLevel);

    return bmr * (activity?.factor ?? 1.55);
  }

  function getTone(calories) {
    if (calories < 100) return 'green';
    if (calories <= 250) return 'yellow';
    if (calories <= 500) return 'orange';
    if (calories <= 750) return 'red';
    return 'purple';
  }

  function weightUnit() {
    return profile?.units === 'metric' ? 'kg' : 'lb';
  }

  function displayWeight(weightLb) {
    const value = profile?.units === 'metric' ? weightLb / LB_PER_KG : weightLb;
    return `${value.toFixed(1)} ${weightUnit()}`;
  }

  function displayDifference(weightLb) {
    const value = profile?.units === 'metric' ? weightLb / LB_PER_KG : weightLb;
    const sign = value > 0 ? '+' : '';
    return `${sign}${value.toFixed(2)} ${weightUnit()}`;
  }

  function formatDate(dateKey) {
    return new Date(`${dateKey}T12:00:00`).toLocaleDateString(undefined, {
      month: 'short',
      day: 'numeric',
      year: 'numeric'
    });
  }
</script>

<svelte:head>
  <meta name="theme-color" content={tone === 'purple' ? '#6d28d9' : '#0f766e'} />
</svelte:head>

<main class={`app tone-${tone}`}>
  <header class="topbar">
    <div>
      <p class="eyebrow">Maintainer</p>
      <h1>{profile ? 'Daily Weight Check' : 'Set Your Baseline'}</h1>
    </div>

    {#if profile}
      <nav class="tabs" aria-label="Main navigation">
        <button class:active={screen === 'record'} onclick={() => (screen = 'record')}>Record</button>
        <button class:active={screen === 'history'} onclick={() => (screen = 'history')}>History</button>
        <button class:active={screen === 'settings'} onclick={() => (screen = 'settings')}>Settings</button>
      </nav>
    {/if}
  </header>

  {#if !profile}
    <section class="panel setup-panel" aria-labelledby="setup-title">
      <div>
        <p class="section-kicker">No login needed</p>
        <h2 id="setup-title">Tell Maintainer how to calculate your day.</h2>
      </div>
      <form class="form-grid" onsubmit={saveProfile}>
        <fieldset class="segmented">
          <legend>Units</legend>
          <label>
            <input bind:group={profileForm.units} type="radio" value="imperial" />
            <span>Imperial</span>
          </label>
          <label>
            <input bind:group={profileForm.units} type="radio" value="metric" />
            <span>Metric</span>
          </label>
        </fieldset>

        <label>
          <span>Year of birth</span>
          <input bind:value={profileForm.birthYear} inputmode="numeric" placeholder="1990" />
        </label>

        <label>
          <span>Gender</span>
          <select bind:value={profileForm.gender}>
            <option value="male">Male</option>
            <option value="female">Female</option>
          </select>
        </label>

        {#if profileForm.units === 'imperial'}
          <div class="inline-fields">
            <label>
              <span>Height</span>
              <input bind:value={profileForm.heightFeet} inputmode="numeric" placeholder="5" />
              <small>ft</small>
            </label>
            <label>
              <span>&nbsp;</span>
              <input bind:value={profileForm.heightInches} inputmode="decimal" placeholder="10" />
              <small>in</small>
            </label>
          </div>
        {:else}
          <label>
            <span>Height</span>
            <input bind:value={profileForm.heightCm} inputmode="decimal" placeholder="178" />
            <small>cm</small>
          </label>
        {/if}

        <label>
          <span>Activity level</span>
          <select bind:value={profileForm.activityLevel}>
            {#each activityLevels as level}
              <option value={level.value}>{level.label}</option>
            {/each}
          </select>
        </label>

        {#if setupError}
          <p class="error" role="alert">{setupError}</p>
        {/if}

        <button class="primary-action" type="submit">Start Maintaining</button>
      </form>
    </section>
  {:else if screen === 'record'}
    <section class="dashboard">
      <div class="panel record-panel">
        <div>
          <p class="section-kicker">Morning check-in</p>
          <h2>Record your bodyweight each morning under the same conditions.</h2>
        </div>

        <form class="weight-form" onsubmit={saveWeight}>
          <label for="weight">Bodyweight</label>
          <div class="weight-row">
            <input
              id="weight"
              bind:value={weightInput}
              inputmode="decimal"
              placeholder={profile.units === 'metric' ? '82.5' : '182.0'}
              aria-describedby="weight-unit"
            />
            <span id="weight-unit">{weightUnit()}</span>
            <button type="submit">Submit</button>
          </div>
          {#if todayEntry}
            <p class="hint">Today is recorded as {displayWeight(todayEntry.weightLb)}.</p>
          {/if}
          {#if weightError}
            <p class="error" role="alert">{weightError}</p>
          {/if}
        </form>
      </div>

      <div class="panel result-panel">
        {#if !analysis.ready}
          <p class="section-kicker">{analysis.total} of 4 days recorded</p>
          <h2>Record your bodyweight for another {analysis.daysNeeded} {analysis.daysNeeded === 1 ? 'day' : 'days'} to establish a baseline.</h2>
        {:else}
          <p class="section-kicker">Rolling average vs baseline average</p>
          <div class="difference">{displayDifference(analysis.differenceLb)}</div>
          <div class="metric-grid">
            <div>
              <span>Rolling average</span>
              <strong>{displayWeight(analysis.rollingAverageLb)}</strong>
            </div>
            <div>
              <span>Baseline average</span>
              <strong>{displayWeight(analysis.baselineAverageLb)}</strong>
            </div>
            <div>
              <span>Sample window</span>
              <strong>{analysis.sampleDays} days</strong>
            </div>
            <div>
              <span>TDEE</span>
              <strong>{Math.round(analysis.tdee)} cal</strong>
            </div>
          </div>

          {#if analysis.differenceLb > 0}
            <p class="recommendation">
              About {Math.round(analysis.calorieDifference)} calories above baseline. Eat about
              <strong>{Math.round(analysis.recommendedCalories)} calories</strong>
              today to return to the baseline average.
            </p>
          {:else}
            <p class="recommendation">
              Your rolling average is at or below baseline. Eat about
              <strong>{Math.round(analysis.tdee)} calories</strong>
              to maintain today.
            </p>
          {/if}
        {/if}
      </div>
    </section>
  {:else if screen === 'history'}
    <section class="panel history-panel">
      <div>
        <p class="section-kicker">{sortedEntries.length} total entries</p>
        <h2>Recorded Weights</h2>
      </div>

      {#if sortedEntries.length === 0}
        <p class="empty-state">No bodyweight entries yet.</p>
      {:else}
        <div class="table-wrap">
          <table>
            <thead>
              <tr>
                <th>Date</th>
                <th>Weight</th>
              </tr>
            </thead>
            <tbody>
              {#each [...sortedEntries].reverse() as entry}
                <tr>
                  <td>{formatDate(entry.date)}</td>
                  <td>{displayWeight(entry.weightLb)}</td>
                </tr>
              {/each}
            </tbody>
          </table>
        </div>
      {/if}
    </section>
  {:else if screen === 'settings'}
    <section class="settings-layout">
      <div class="panel settings-panel">
        <div>
          <p class="section-kicker">Settings</p>
          <h2>Edit Your Information</h2>
        </div>
        <form class="form-grid" onsubmit={saveProfile}>
          <fieldset class="segmented">
            <legend>Units</legend>
            <label>
              <input bind:group={profileForm.units} type="radio" value="imperial" />
              <span>Imperial</span>
            </label>
            <label>
              <input bind:group={profileForm.units} type="radio" value="metric" />
              <span>Metric</span>
            </label>
          </fieldset>

          <label>
            <span>Year of birth</span>
            <input bind:value={profileForm.birthYear} inputmode="numeric" />
          </label>

          <label>
            <span>Gender</span>
            <select bind:value={profileForm.gender}>
              <option value="male">Male</option>
              <option value="female">Female</option>
            </select>
          </label>

          {#if profileForm.units === 'imperial'}
            <div class="inline-fields">
              <label>
                <span>Height</span>
                <input bind:value={profileForm.heightFeet} inputmode="numeric" />
                <small>ft</small>
              </label>
              <label>
                <span>&nbsp;</span>
                <input bind:value={profileForm.heightInches} inputmode="decimal" />
                <small>in</small>
              </label>
            </div>
          {:else}
            <label>
              <span>Height</span>
              <input bind:value={profileForm.heightCm} inputmode="decimal" />
              <small>cm</small>
            </label>
          {/if}

          <label>
            <span>Activity level</span>
            <select bind:value={profileForm.activityLevel}>
              {#each activityLevels as level}
                <option value={level.value}>{level.label}</option>
              {/each}
            </select>
          </label>

          {#if setupError}
            <p class="error" role="alert">{setupError}</p>
          {/if}

          <button class="primary-action" type="submit">Save Settings</button>
        </form>
      </div>

      <div class="panel reset-panel">
        <p class="section-kicker">Reset</p>
        <h2>Clear all local data</h2>
        <p>This removes your profile and recorded weights from this browser.</p>
        {#if resetConfirming}
          <div class="reset-confirmation" role="group" aria-label="Confirm reset">
            <p>Are you sure?</p>
            <div class="reset-actions">
              <button class="secondary-action" type="button" onclick={cancelReset}>Cancel</button>
              <button class="danger-action" type="button" onclick={resetApp}>Reset</button>
            </div>
          </div>
        {:else}
          <button class="danger-action" type="button" onclick={confirmReset}>Reset App</button>
        {/if}
      </div>
    </section>
  {/if}
</main>
