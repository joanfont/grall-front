<script lang='ts'>
  import AutoComplete from 'simple-svelte-autocomplete';
  import { onMount } from 'svelte';

  import { allTracks, currentAttempt, currentSong, pastAttempts, songPaused, temporaryAttempt } from '$src/store';
  import { convertSpotifyTrackToSong } from '$lib/util';
  import type { Attempt, Guess, Song, SpotifyTrack } from '$src/types';
  import Button from '$components/Button.svelte';
  import GameEnd from '$components/GameEnd.svelte';
  import analytics from '$lib/analytics';
  import { toast } from '@zerodevx/svelte-toast';

  export let random = false;
  let attempt = <Attempt>{};
  export let custom = false;

  // TODO: rewrite store logic to support storing multiple attempts (and maybe custom playlists)
  //       cuz it's hideous right now
  onMount(async () => {
    // if we are in a new date from the past, take the new random song and set it to the current one.
    //    reset the attempts.
    if (
      new Date(currentAttempt.get().date).toLocaleDateString() !==
      new Date().toLocaleDateString() ||
      !currentAttempt
    ) {
      currentAttempt.set(<Attempt>{
        guesses: [],
        date: new Date(),
        correct: false,
        attempts: 0
      });
    }
    if (custom || random) {
      attempt = temporaryAttempt.get();
      temporaryAttempt.listen((value) => (attempt = value));
    } else {
      attempt = currentAttempt.get();
      currentAttempt.listen((value) => (attempt = value));
    }
    if (random && custom) temporaryAttempt.setKey('type', 'custom_random');
    else if (random) temporaryAttempt.setKey('type', 'random');
    else if (custom) temporaryAttempt.setKey('type', 'custom');
    else currentAttempt.setKey('type', 'default');
  });
  let currentSelectedSong = <Song>{};

  const chooseSong = () => {
    if (!currentSelectedSong || !currentSelectedSong.id) {
      toast.push('Please select a valid song from the list.', {
        theme: {
          '--toastWidth': '20rem',
          '--toastBackground': '#e64949',
          '--toastBarBackground': '#C53030'
        }
      });
      return;
    }
    const guesses = [...(attempt.guesses || [])];
    if (currentSelectedSong.id === currentSong.get().id) {
      guesses.push(<Guess>{
        song: currentSong.get(),
        correct: true,
        artistCorrect: true
      });
      if (custom || random) {
        temporaryAttempt.setKey('guesses', guesses);
        temporaryAttempt.setKey('correct', true);
        temporaryAttempt.setKey('attempts', attempt.attempts + 1);
      } else {
        currentAttempt.setKey('guesses', guesses);
        currentAttempt.setKey('correct', true);
        currentAttempt.setKey('attempts', attempt.attempts + 1);
      }
      analytics.track('guess-song', { correct: true, custom, random: random });
      analytics.track('attempt-correct', { custom, random: random });
      // save correct attempt to localstorage
      pastAttempts.set({ array: [...pastAttempts.get().array, attempt] });
    } else {
      const track = $allTracks.find((t) => t.id == currentSelectedSong.id);
      guesses.push(<Guess>{
        song: convertSpotifyTrackToSong(track),
        correct: false,
        artistCorrect: currentSelectedSong.artist.includes(currentSong.get().artist)
      });
      if (custom || random) {
        temporaryAttempt.setKey('guesses', guesses);
        temporaryAttempt.setKey('attempts', attempt.attempts + 1);
      } else {
        currentAttempt.setKey('guesses', guesses);
        currentAttempt.setKey('attempts', attempt.attempts + 1);
      }
      analytics.track('guess-song', { correct: false, custom, random: random });
    }
    if (attempt.attempts === 6 && !attempt.correct) {
      analytics.track('attempt-fail', { custom, random: random });
      // save incorrect attempt to localstorage
      pastAttempts.set({ array: [...pastAttempts.get().array, attempt] });
    }
    currentSelectedSong = undefined;
    songPaused.set(true);
  };

  const skipSong = () => {
    const guesses = [...(attempt.guesses || [])];
    guesses.push(<Guess>{
      song: null,
      correct: false,
      artistCorrect: false
    });
    if (custom || random) {
      temporaryAttempt.setKey('guesses', guesses);
      temporaryAttempt.setKey('attempts', attempt.attempts + 1);
    } else {
      currentAttempt.setKey('guesses', guesses);
      currentAttempt.setKey('attempts', attempt.attempts + 1);
    }
    analytics.track('skip-song', { custom, random: random });
  };

  const searchSongs = async (query: string) => {
    let searchResults: SpotifyTrack[] | Song[] = $allTracks.filter((t) => {
      return (t.name + ' ' + t.artists[0].name).toLowerCase().includes(query.toLowerCase());
    });
    searchResults = searchResults.map((t) => convertSpotifyTrackToSong(t));
    searchResults = searchResults.map((s) => <Song>{
      name: s.name + ' by ' + s.artist,
      id: s.id,
      artist: s.artist,
      preview: s.preview
    });
    return searchResults;
  };
</script>

<div>
  <!-- PLAYLIST/GENRE TITLE -->
  {#if attempt.attempts === 0}
    <div class='w-full px-0 sm:px-20 transition-all duration-200'>
      <span class='text-center mx-auto w-full text-blue-100'
      >listen to the song and try to guess it correctly. you have 6 attempts.</span
      >
    </div>
  {/if}
  <!-- GUESSES -->
  <div class='w-full px-0 transition-all sm:px-20 items-center game'>
    {#if attempt.guesses}
      {#each attempt.guesses.filter((g) => !g.correct) as guess}
        {#if guess.song}
          <div
            title='Open in Spotify'
            on:click={() => {
              window.open(`https://open.spotify.com/track/${guess.song.id}`, '_blank').focus();
            }}
            class={`cursor-pointer ${
              guess.artistCorrect ? 'border-amber-400' : 'border-red-600'
            } border-2 h-10 p-2 my-2 w-full text-left rounded-sm bg-gray-900 overflow-ellipsis whitespace-nowrap overflow-hidden`}
          >
            {guess.song.name} by {guess.song.artist}
          </div>
        {:else}
          <div
            class={`border-gray-600 border-2 h-10 p-2 my-2 w-full text-left rounded-sm bg-gray-900 overflow-ellipsis whitespace-nowrap overflow-hidden`}
          >
            song guess skipped
          </div>
        {/if}
      {/each}
      {#each attempt.guesses.filter((g) => g.correct) as guess}
        <div
          title='Open in Spotify'
          on:click={() => {
            window.open(`https://open.spotify.com/track/${guess.song.id}`, '_blank').focus();
          }}
          class='cursor-pointer border-green-600 border-2 h-10 p-2 my-2 w-full text-left rounded-sm bg-gray-900 overflow-ellipsis whitespace-nowrap overflow-hidden'
        >
          {guess.song.name} by {guess.song.artist}
        </div>
      {/each}
    {/if}
    {#if attempt.attempts < 6 && !attempt.correct}
      <div class='flex mt-6 mb-2' title='guess a song'>
        <AutoComplete
          name='song-selection'
          className='w-10/12'
          inputClassName='border-gray-600 border-2 w-full h-8 px-2 py-5 rounded-sm bg-gray-900 hover:border-gray-400 focus:border-gray-400 outline-none transition-all duration-200'
          dropdownClassName='p-0 bg-gray-900'
          placeholder={`${6 - attempt.attempts} ${
            6 - attempt.attempts !== 1 ? 'attempts' : 'attempt'
          } left`}
          minCharactersToSearch={2}
          searchFunction={searchSongs}
          bind:selectedItem={currentSelectedSong}
          labelFieldName='name'
          valueFieldName='id'
          showLoadingIndicator
          noInputStyles
          hideArrow
        >
          <div
            slot='item'
            let:item
            class='border-2 h-10 px-2 py-3 w-full text-left rounded-sm bg-gray-900 text-white hover:text-blue-500 hover:border-blue-500 overflow-ellipsis whitespace-nowrap overflow-hidden transition-colors duration-150'
          >
            <span>{item.name}</span>
          </div>
          <div slot='no-results' class='py-1'>
            <span>could not find this song in the playlist.</span>
          </div>
          <div slot='loading' class='py-1'>
            <span>searching for songs...</span>
          </div>
        </AutoComplete>
        <div class='w-2/12 pl-4 mt-0.5' title='guess selected song'>
          <Button title='Submit Song Guess' type='primary' className='w-full' on:click={chooseSong}>
            <svg
              xmlns='http://www.w3.org/2000/svg'
              class='h-6 w-6 mx-auto'
              fill='none'
              viewBox='0 0 24 24'
              stroke='currentColor'
              stroke-width='2'
            >
              <path stroke-linecap='round' stroke-linejoin='round' d='M13 7l5 5m0 0l-5 5m5-5H6' />
            </svg>
          </Button>
          <div
            class='text-gray-400 cursor-pointer text-center underline underline-offset-1'
            on:click={skipSong}
          >
            skip
          </div>
        </div>
      </div>
    {:else}
      <GameEnd custom={custom} {random} />
    {/if}
  </div>
</div>
