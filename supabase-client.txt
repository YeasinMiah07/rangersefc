// ═══════════════════════════════════════════════════════════════
// Rangers Efootball Club — Supabase Client & Data Helpers
// Include this file on every page via:
//   <script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2/dist/umd/supabase.min.js"></script>
//   <script src="supabase-client.js"></script>
// ═══════════════════════════════════════════════════════════════

// ── STEP 1: FILL THESE IN ──
// Get these from Supabase Dashboard → Project Settings → API
const SUPABASE_URL = "https://qucnqmrjtmzjasgdicgg.supabase.co";
const SUPABASE_ANON_KEY = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InF1Y25xbXJqdG16amFzZ2RpY2dnIiwicm9sZSI6ImFub24iLCJpYXQiOjE3ODU5Njg2MTIsImV4cCI6MjEwMTU0NDYxMn0.xIVRaqfBVa0Sg4sDiDSJ4TLwSoEWu2alsdn9ZUpENO4";

const supabase = window.supabase.createClient(SUPABASE_URL, SUPABASE_ANON_KEY);

// ═══════════════════════════════════════════════════════════════
// AUTH — Signup, Login, Logout, Session
// ═══════════════════════════════════════════════════════════════

// Public signup: creates a normal fan account (role defaults to 'public')
async function signUpPublic(email, password, displayName, username){
  const { data, error } = await supabase.auth.signUp({
    email, password,
    options: { data: { display_name: displayName, username: username || null } }
  });
  if(error) return { ok:false, error: error.message };
  // If Supabase has "Confirm email" turned ON, signUp() does NOT create
  // an active session — data.session will be null until they click the
  // email link. Callers need to know this to avoid silently failing
  // the role claim step below (which requires an active session).
  const hasSession = !!data.session;
  return { ok:true, user: data.user, hasSession };
}

// Player signup: creates account, then immediately tries to claim the
// 'player' role using the access code + which roster row is theirs
async function signUpPlayer(email, password, displayName, username, accessCode, playerId){
  const signup = await signUpPublic(email, password, displayName, username);
  if(!signup.ok) return signup;
  if(!signup.hasSession){
    return { ok:false, needsEmailConfirm:true, error:
      'Account created — but your club\'s Supabase project requires email confirmation first. Check your email, click the confirm link, then log in and ask an admin to link your player role (or turn off "Confirm email" in Supabase → Authentication → Settings for instant signup).' };
  }
  const claim = await supabase.rpc('claim_role', {
    input_code: accessCode, target_role: 'player', target_player_id: playerId
  });
  if(claim.error || claim.data !== true){
    return { ok:false, error: 'Account created, but the player access code was incorrect. Contact a club admin to link your account.' };
  }
  return { ok:true };
}

// Admin signup: creates account, then claims 'admin' role using the admin code
async function signUpAdmin(email, password, displayName, username, accessCode){
  const signup = await signUpPublic(email, password, displayName, username);
  if(!signup.ok) return signup;
  if(!signup.hasSession){
    return { ok:false, needsEmailConfirm:true, error:
      'Account created — but your club\'s Supabase project requires email confirmation first. Check your email, click the confirm link, then log in and try claiming admin again (or turn off "Confirm email" in Supabase → Authentication → Settings for instant signup).' };
  }
  const claim = await supabase.rpc('claim_role', {
    input_code: accessCode, target_role: 'admin', target_player_id: null
  });
  if(claim.error || claim.data !== true){
    return { ok:false, error: 'Account created, but the admin access code was incorrect.' };
  }
  return { ok:true };
}

async function signIn(email, password){
  const { data, error } = await supabase.auth.signInWithPassword({ email, password });
  if(error) return { ok:false, error: error.message };
  return { ok:true, user: data.user };
}

async function signOut(){
  await supabase.auth.signOut();
}

// ─── PASSWORD RESET ───
// Step 1: user requests a reset link by email
async function requestPasswordReset(email){
  const { error } = await supabase.auth.resetPasswordForEmail(email, {
    redirectTo: window.location.origin + '/reset-password.html'
  });
  if(error) return { ok:false, error: error.message };
  return { ok:true };
}
// Step 2: user arrives at reset-password.html via the emailed link
// (Supabase puts them in a temporary logged-in state) and sets a new password
async function setNewPassword(newPassword){
  const { error } = await supabase.auth.updateUser({ password: newPassword });
  if(error) return { ok:false, error: error.message };
  return { ok:true };
}

async function getCurrentProfile(){
  const { data: { session } } = await supabase.auth.getSession();
  if(!session) return null;
  const { data, error } = await supabase.from('profiles').select('*').eq('id', session.user.id).single();
  if(error) return null;
  return data;
}

// Call this at the top of any admin-only page. Redirects to auth.html
// if not logged in, or if logged in but not an admin.
async function requireAdmin(){
  const profile = await getCurrentProfile();
  if(!profile || profile.role !== 'admin'){
    window.location.href = 'auth.html?redirect=' + encodeURIComponent(window.location.pathname);
    return null;
  }
  return profile;
}

// ═══════════════════════════════════════════════════════════════
// SELF-SERVICE PROFILE EDITING
// Both go through narrow Postgres functions — the client never gets
// to touch role/player_id/rating/goals/mvps directly.
// ═══════════════════════════════════════════════════════════════
async function updateDisplayName(newName){
  const { error } = await supabase.rpc('update_display_name', { new_name: newName });
  if(error){ console.error('updateDisplayName error:', error); return false; }
  return true;
}

async function updateUsername(newUsername){
  const { data, error } = await supabase.rpc('update_username', { new_username: newUsername });
  if(error){ console.error('updateUsername error:', error); return 'error'; }
  return data; // 'ok' or 'taken'
}

// Uploads a file to the 'avatars' storage bucket, named after the
// user's own ID (required by the storage RLS policy), then saves the
// public URL to their profile (and linked player row, if any).
async function uploadAvatar(file){
  const { data: { session } } = await supabase.auth.getSession();
  if(!session) return { ok:false, error:'Not logged in' };
  const ext = file.name.split('.').pop();
  const path = `${session.user.id}/avatar.${ext}`;
  const { error: uploadError } = await supabase.storage.from('avatars').upload(path, file, { upsert: true });
  if(uploadError) return { ok:false, error: uploadError.message };
  const { data: urlData } = supabase.storage.from('avatars').getPublicUrl(path);
  const publicUrl = urlData.publicUrl;
  const { error: saveError } = await supabase.rpc('update_avatar', { new_url: publicUrl });
  if(saveError) return { ok:false, error: saveError.message };
  return { ok:true, url: publicUrl };
}

async function updatePlayerStyle(newStyle, newBio){
  const { data, error } = await supabase.rpc('update_player_style', { new_style: newStyle, new_bio: newBio });
  if(error){ console.error('updatePlayerStyle error:', error); return false; }
  return data === true;
}

async function fetchLinkedPlayer(playerId){
  if(!playerId) return null;
  const { data, error } = await supabase.from('players').select('*').eq('id', playerId).single();
  if(error){ console.error('fetchLinkedPlayer error:', error); return null; }
  return data;
}

// ═══════════════════════════════════════════════════════════════
// TOURNAMENT HISTORY (shown on a player's public profile)
// ═══════════════════════════════════════════════════════════════
async function fetchPlayerHistory(playerId){
  const { data, error } = await supabase.from('player_tournament_history')
    .select('*').eq('player_id', playerId).order('event_date', { ascending: false });
  if(error){ console.error('fetchPlayerHistory error:', error); return []; }
  return data;
}

// Admin-only in practice (RLS blocks non-admins from inserting)
async function addPlayerHistory(playerId, tournamentName, result, eventDate){
  const { error } = await supabase.from('player_tournament_history').insert([{
    player_id: playerId, tournament_name: tournamentName, result, event_date: eventDate || new Date().toISOString().slice(0,10)
  }]);
  if(error){ console.error('addPlayerHistory error:', error); return false; }
  return true;
}

// ═══════════════════════════════════════════════════════════════
// TOURNAMENT SIGNUPS (players join an upcoming/live tournament)
// ═══════════════════════════════════════════════════════════════
async function joinTournament(tournamentId){
  const profile = await getCurrentProfile();
  if(!profile) return { ok:false, reason:'not-logged-in' };
  if(!profile.player_id) return { ok:false, reason:'not-a-player' };
  const { error } = await supabase.from('tournament_signups').insert([{
    tournament_id: tournamentId, player_id: profile.player_id
  }]);
  if(error){
    if(error.code === '23505') return { ok:false, reason:'already-joined' };
    console.error('joinTournament error:', error);
    return { ok:false, reason:'error' };
  }
  return { ok:true };
}

async function fetchTournamentSignups(tournamentId){
  const { data, error } = await supabase.from('tournament_signups')
    .select('*, players(name, position, rating)').eq('tournament_id', tournamentId);
  if(error){ console.error('fetchTournamentSignups error:', error); return []; }
  return data;
}

// ═══════════════════════════════════════════════════════════════
// PLAYERS
// ═══════════════════════════════════════════════════════════════
async function fetchPlayers(){
  const { data, error } = await supabase.from('players').select('*').order('rating', { ascending: false });
  if(error){ console.error('fetchPlayers error:', error); return []; }
  return data;
}

async function addPlayerSB({name, position, category, rating, style}){
  const { data, error } = await supabase.from('players').insert([{
    name, position, category, rating: parseInt(rating) || 75, style, number: null, form: []
  }]).select();
  if(error){ console.error('addPlayer error:', error); return null; }
  return data[0];
}

// ═══════════════════════════════════════════════════════════════
// NEWS
// ═══════════════════════════════════════════════════════════════
async function fetchNews(){
  const { data, error } = await supabase.from('news').select('*').order('created_at', { ascending: false });
  if(error){ console.error('fetchNews error:', error); return []; }
  return data;
}

async function addNewsPost({title, category, content}){
  const { data, error } = await supabase.from('news').insert([{ title, category, content }]).select();
  if(error){ console.error('addNewsPost error:', error); return null; }
  return data[0];
}

// ═══════════════════════════════════════════════════════════════
// VOTES (MVP voting — requires a logged-in account, any role)
// ═══════════════════════════════════════════════════════════════
async function fetchVoteCounts(){
  const { data, error } = await supabase.from('votes').select('player_id');
  if(error){ console.error('fetchVoteCounts error:', error); return {}; }
  const counts = {};
  data.forEach(v => { counts[v.player_id] = (counts[v.player_id] || 0) + 1; });
  return counts;
}

async function castVote(playerId){
  const { data: { session } } = await supabase.auth.getSession();
  if(!session){
    return { ok:false, reason:'not-logged-in' };
  }
  const { error } = await supabase.from('votes').insert([{ player_id: playerId, voter_id: session.user.id }]);
  if(error){
    if(error.code === '23505') return { ok:false, reason:'already-voted' };
    console.error('castVote error:', error);
    return { ok:false, reason:'error' };
  }
  return { ok:true };
}

// ═══════════════════════════════════════════════════════════════
// MATCHES
// ═══════════════════════════════════════════════════════════════
async function fetchMatches(){
  const { data, error } = await supabase.from('matches').select('*').order('match_date', { ascending: false });
  if(error){ console.error('fetchMatches error:', error); return []; }
  return data;
}

async function addMatch({home, away, home_score, away_score, tournament_id, status}){
  const { data, error } = await supabase.from('matches').insert([{
    home, away, home_score, away_score, tournament_id, status: status || 'upcoming'
  }]).select();
  if(error){ console.error('addMatch error:', error); return null; }
  return data[0];
}

// ═══════════════════════════════════════════════════════════════
// TOURNAMENTS (replaces the localStorage sync between Hub and Live page)
// ═══════════════════════════════════════════════════════════════
async function fetchLiveTournament(){
  const { data, error } = await supabase.from('tournaments')
    .select('*').eq('status', 'live')
    .order('updated_at', { ascending: false }).limit(1);
  if(error){ console.error('fetchLiveTournament error:', error); return null; }
  return data[0] || null;
}

async function saveTournamentSB(tournamentId, name, format, stateData){
  const payload = { name, format, status: 'live', data: stateData, updated_at: new Date().toISOString() };
  if(tournamentId){
    const { data, error } = await supabase.from('tournaments').update(payload).eq('id', tournamentId).select();
    if(error){ console.error('updateTournament error:', error); return null; }
    return data[0];
  } else {
    const { data, error } = await supabase.from('tournaments').insert([payload]).select();
    if(error){ console.error('insertTournament error:', error); return null; }
    return data[0];
  }
}

async function endTournament(tournamentId){
  const { error } = await supabase.from('tournaments').update({ status: 'completed' }).eq('id', tournamentId);
  if(error) console.error('endTournament error:', error);
}
