
```datacorejsx
// Config
// Vault_name is important, if you wanna use advancedURI to have Links to your Journals
const VAULT_NAME = "obsidianSync";
// This is important where the script should look for the Journals. In the journals the information will be saved in the metadata
const JOURNAL_PATH = "Journal/Tag/2025";
// True: last Journal will be left. False: last journal will be on the right.
const REVERSE_ORDER = true; 
// Here you can define your habits. Following values are possible:
// id: is more for the code just put a text in it.
// emoji: is the icon which will be shown
// Label: is the text, that will be shown
// isBoolean: true. Use this, if you want to have a "just true or false" value. You did Meditate. You don't mind if 5 or 30 minutes. You still will have to set defaultDuration: 1 .
// defaultDuration: x. This is good if you want to set a value for a habit. If you click the button it will always put x in it.
// isincremental: whis will increment/fill up the habit. I use it for drinking - for every glas of water i click one time on the button to log it.
const HABITS = [
  { id: 'Aufstehen', emoji: '🌄', label: 'Getting up at 6 a.m', isBoolean: true, defaultDuration: 1, monthlyGoal: 20 },
  { id: 'Journaling', emoji: '✍️', label: 'Journaling', isBoolean: true, defaultDuration: 1, monthlyGoal: 25 },
  { id: 'Trinken', emoji: '🥛', label: 'Drink  Water', defaultDuration: 8, unit: 'glas', monthlyGoal: 240 },
  { id: 'Yoga', emoji: '🧘‍♂️', label: 'Yoga', isBoolean: true, defaultDuration: 1, monthlyGoal: 20},
  { id: 'angst', emoji: '🧑‍🤝‍🧑', label: 'Write a Friend', isBoolean: true, defaultDuration: 1, monthlyGoal: 20 }
];

const GOALS = {
  perfectDays: {
    monthly: 20,
    yearly: 250
  }
};

const DAYS = ['Sun', 'Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat'];

const COLORS = {
  primary: 'var(--interactive-accent)',
  secondary: 'var(--background-secondary)',
  hoverState: 'var(--interactive-accent-hover)',
  textPrimary: 'var(--text-normal)',
  textLight: 'var(--text-on-accent)',
  progressBar: {
    low: 'var(--interactive-accent)',
    medium: 'var(--interactive-accent)',
    high: 'var(--interactive-accent)',
    gradient: {
      start: 'var(--interactive-accent)',
      end: 'var(--interactive-accent-hover)'
    }
  }
};

// Utility Functions
const formatMetricValue = (value, habit) => {
  if (value === null || value === undefined) return '0';

  switch(habit.unit) {
    case 'minutes':
      return value === 1 ? '1 Minute' : `${value} Minutes`;
    case 'dollars':
      return `$${Number(value).toLocaleString()}`;
    case 'rupees':
      return `Rs. ${Number(value).toLocaleString()}`;
    default:
      return value;
  }
};

function formatDate(date) {
  const yyyy = date.getFullYear();
  const mm = String(date.getMonth() + 1).padStart(2, '0');
  const dd = String(date.getDate()).padStart(2, '0');
  return `${yyyy}-${mm}-${dd}`;
}

function getJournalPath(date) {
  const mm = String(date.getMonth() + 1).padStart(2, '0');
  const day = formatDate(date);
  return `${JOURNAL_PATH}/${mm}/${day}`;
}

function getAdvancedUriLink(date) {
  const path = getJournalPath(date);
  // encode nur spezielle Zeichen, nicht die Slashes
  const encodedPath = path
    .split('/')
    .map(segment => encodeURIComponent(segment))
    .join('/');
  return `obsidian://advanced-uri?vault=${encodeURIComponent(VAULT_NAME)}&filepath=${encodedPath}`;
}

const calculateTrendPercentage = (current, previous) => {
  if (previous === 0) return current > 0 ? 100 : 0;
  return ((current - previous) / previous) * 100;
};

// Moved getCompletionColor to top level
const getCompletionColor = (percentage) => {
  if (percentage >= 75) return COLORS.progressBar.high;
  if (percentage >= 50) return COLORS.progressBar.medium;
  return COLORS.progressBar.low;
};

// Base Components
const CircularProgress = ({ value, size, color = 'var(--interactive-accent)' }) => {
  const strokeWidth = 4;
  const radius = (size - strokeWidth * 2) / 2;
  const circumference = 2 * Math.PI * radius;
  const progress = ((100 - value) / 100) * circumference;

  return (
    <svg
      width={size}
      height={size}
      viewBox={`0 0 ${size} ${size}`}
      style={{
        transform: 'rotate(-90deg)',
        overflow: 'visible'
      }}
    >
      <circle
        cx={size / 2}
        cy={size / 2}
        r={radius}
        stroke="var(--background-modifier-border)"
        strokeWidth={strokeWidth}
        fill="none"
      />
      <circle
        cx={size / 2}
        cy={size / 2}
        r={radius}
        stroke={color}
        strokeWidth={strokeWidth}
        strokeDasharray={circumference}
        strokeDashoffset={progress}
        fill="none"
        style={{
          transition: 'stroke-dashoffset 0.5s ease',
          transformOrigin: 'center'
        }}
      />
    </svg>
  );
};

// Additional Base Components
const TrendIndicator = ({ current, previous }) => {
  const trend = calculateTrendPercentage(current, previous);
  let color = 'var(--text-normal)';
  let indicator = '→';

  if (trend > 0) {
    color = 'var(--color-green)';
    indicator = '↑';
  } else if (trend < 0) {
    color = 'var(--color-red)';
    indicator = '↓';
  }

  return (
    <span style={{ color }}>
      {indicator} {Math.abs(trend).toFixed(1)}%
    </span>
  );
};

const TimeInput = ({
  entry,
  habitId,
  editingTime,
  setEditingTime,
  updateHabitDuration,
  getHabitStatus,
  getHabitDuration
}) => {
  const habit = HABITS.find(h => h.id === habitId);
  if (habit.isBoolean) return null;
  const duration = getHabitDuration(entry, habitId);
  const isEditing = editingTime?.entryPath === entry.$path && editingTime?.habitId === habitId;

  if (!getHabitStatus(entry, habitId)) return null;

  if (isEditing) {
    return (
      <input
        type="number"
        defaultValue={duration}
        min="0"
        style={{
          width: '60px',
          padding: '2px',
          fontSize: '0.9em',
          textAlign: 'center'
        }}
        onBlur={(e) => updateHabitDuration(entry, habitId, e.target.value)}
        autoFocus
      />
    );
  }

  return (
    <span
      onClick={() => setEditingTime({ entryPath: entry.$path, habitId })}
      style={{ cursor: 'pointer', fontSize: '0.8em' }}
    >
      {formatMetricValue(duration, HABITS.find(h => h.id === habitId))}
    </span>
  );
};

const ProgressBar = ({ value, max, color = 'var(--interactive-accent)' }) => {
  const percentage = Math.min((value / max) * 100, 100);
  return (
    <div style={{
      width: '100%',
      height: '4px',
      backgroundColor: 'var(--background-modifier-border)',
      borderRadius: '4px',
      overflow: 'hidden'
    }}>
      <div style={{
        width: `${percentage}%`,
        height: '100%',
        backgroundColor: color,
        transition: 'width 0.3s ease'
      }} />
    </div>
  );
};

const StyledCard = ({ children, extraStyles = {} }) => (
  <div style={{
    backgroundColor: 'var(--background-primary)',
    borderRadius: '12px',
    boxShadow: '0 2px 8px rgba(0, 0, 0, 0.1)',
    padding: '24px',
    transition: 'all 0.2s ease',
    ':hover': {
      boxShadow: '0 4px 12px rgba(0, 0, 0, 0.15)',
      transform: 'translateY(-2px)'
    },
    ...extraStyles
  }}>
    {children}
  </div>
);

const ActionButton = ({ icon, label, onClick, isActive, extraStyles = {} }) => (
  <button
    onClick={onClick}
    style={{
      padding: '12px 24px',
      borderRadius: '8px',
      border: 'none',
      backgroundColor: isActive ? COLORS.primary : COLORS.secondary,
      color: isActive ? COLORS.textLight : COLORS.textPrimary,
      display: 'flex',
      alignItems: 'center',
      gap: '8px',
      cursor: 'pointer',
      transition: 'all 0.2s ease',
      fontSize: '16px',
      fontWeight: '500',
      ':hover': {
        transform: 'translateY(-1px)',
        backgroundColor: COLORS.primary,
        color: COLORS.textLight,
        boxShadow: '0 2px 4px rgba(0, 0, 0, 0.2)'
      },
      ...extraStyles
    }}
  >
    <span style={{ fontSize: '20px' }}>{icon}</span>
    {label && <span>{label}</span>}
  </button>
);

const NavigationControls = ({
  selectedDate,
  navigateDate,
  activeView,
  setActiveView
}) => (
  <div style={{
    display: 'flex',
    flexDirection: 'column',
    gap: '24px'
  }}>
    <div style={{
      display: 'flex',
      gap: '16px',
      alignItems: 'center',
      justifyContent: 'center',
      background: COLORS.secondary,
      padding: '8px 16px',
      borderRadius: '12px',
      boxShadow: 'var(--shadow-s)'
    }}>
      <ActionButton
        icon="←"
        onClick={() => navigateDate(-1)}
        extraStyles={{
          backgroundColor: COLORS.primary,
          color: COLORS.textLight
        }}
      />
      <div style={{
        fontWeight: 'bold',
        fontSize: '24px',
        minWidth: '240px',
        textAlign: 'center',
        fontFamily: 'var(--font-interface)',
        background: 'var(--background-primary)',
        padding: '8px 16px',
        borderRadius: '8px',
        boxShadow: 'var(--shadow-s)'
      }}>
        {selectedDate.toFormat('MMMM dd, yyyy')}
      </div>
      <ActionButton
        icon="→"
        onClick={() => navigateDate(1)}
        extraStyles={{
          backgroundColor: COLORS.primary,
          color: COLORS.textLight
        }}
      />
    </div>
  </div>
);

const CalendarView = ({
  selectedDate,
  sortedNotes,
  getHabitStatus,
  calculateCompletedHabits,
  updateHabit,
  getHabitDuration,
  editingTime,
  setEditingTime,
  updateHabitDuration
}) => {
  const dates = [];
  let currentDate = selectedDate;

  // Show last 7 days (today and previous 6 days)
  for (let i = 0; i < 7; i++) {
    dates.push(currentDate.minus({ days: i }));
  }

  const notesMap = new Map(sortedNotes.map(note => [note.$name, note]));
  const today = dc.luxon.DateTime.now().startOf('day');

  return (
    <div style={{
      width: '100%',
      overflow: 'hidden',
      padding: '8px 4px'
    }}>
      <div style={{
        display: 'flex',
        gap: '12px',
        overflowX: 'auto',
        paddingBottom: '12px',
        flexDirection: REVERSE_ORDER ? 'row-reverse' : 'row',
        WebkitOverflowScrolling: 'touch',
        scrollbarWidth: 'none',
        msOverflowStyle: 'none',
        '::-webkit-scrollbar': {
          display: 'none'
        }
      }}>
        {dates.map((date) => {
          const dateStr = date.toFormat('yyyy-MM-dd');
          const entry = notesMap.get(dateStr);
          const isSelected = date.hasSame(selectedDate, 'day');
          const isToday = date.hasSame(today, 'day');

          return (
            <div style={{
              flex: '0 0 320px',
              maxWidth: '320px'
            }}>
              <CalendarDayCard
                key={dateStr}
                date={date}
                entry={entry}
                getHabitStatus={getHabitStatus}
                calculateCompletedHabits={calculateCompletedHabits}
                isSelected={isSelected}
                isToday={isToday}
                updateHabit={updateHabit}
                getHabitDuration={getHabitDuration}
                editingTime={editingTime}
                setEditingTime={setEditingTime}
                updateHabitDuration={updateHabitDuration}
              />
            </div>
          );
        })}
      </div>
    </div>
  );
};

const CalendarDayCard = ({
  date,
  entry,
  getHabitStatus,
  calculateCompletedHabits,
  isSelected,
  isToday,
  updateHabit,
  getHabitDuration,
  editingTime,
  setEditingTime,
  updateHabitDuration
}) => {
  const completedCount = calculateCompletedHabits(entry);
  const completionPercentage = entry ? Math.round((completedCount / HABITS.length) * 100) : 0;
  const journalLink = getAdvancedUriLink(date.toJSDate());
  

  return (
	
    <div style={{
      padding: '12px',
      borderRadius: '16px',
      backgroundColor: 'var(--background-primary)',
      color: COLORS.textPrimary,
      boxShadow: 'var(--shadow-s)',
      border: isSelected ? `2px solid ${COLORS.primary}` : '1px solid var(--background-modifier-border)',
      transition: 'all 0.3s cubic-bezier(0.4, 0, 0.2, 1)',
      display: 'flex',
      flexDirection: 'column',
      gap: '8px',
      minHeight: '168px'
    }}>
    <a href={journalLink} target="_blank" rel="noopener noreferrer" style={{ textDecoration: 'none' }}>
      <div style={{
        display: 'flex',
        justifyContent: 'space-between',
        alignItems: 'center',
        backgroundColor: COLORS.secondary,
        padding: '8px 12px',
        borderRadius: '10px'
      }}>
      
        <span style={{
          fontSize: '1em',
          fontWeight: '600',
          color: COLORS.textPrimary
        }}>
          {DAYS[date.weekday % 7]}
        </span>
        <span style={{
          fontWeight: '500',
          fontSize: '0.9em',
          color: COLORS.textPrimary
        }}>
          {date.toFormat('MM-dd')}
        </span>
      </div>
      </a>
	

      {entry && (
        <>
          <div style={{
            display: 'grid',
            gridTemplateColumns: 'repeat(auto-fit, minmax(120px, 1fr))',
            gap: '8px',
            flex: 1,
            padding: '2px'
          }}>
            {HABITS.map(habit => {
              const isCompleted = getHabitStatus(entry, habit.id);
              const duration = getHabitDuration(entry, habit.id);

              return (
                <div
                  key={habit.id}
                  onClick={() => updateHabit(entry, habit.id)}
                  style={{
                    display: 'flex',
                    flexDirection: 'column',
                    alignItems: 'center',
                    justifyContent: 'center',
                    gap: '4px',
                    backgroundColor: isCompleted ? COLORS.primary : COLORS.secondary,
                    borderRadius: '10px',
                    cursor: 'pointer',
                    padding: '12px',
                    width: '100%',
                    minHeight: '80px',
                    transition: 'all 0.2s ease'
                  }}
                >
                  <span style={{
                    fontSize: '24px',
                    marginBottom: '4px'
                  }}>
                    {habit.emoji}
                  </span>
                  <span style={{
                    fontSize: '0.95em',
                    fontWeight: '600',
                    color: isCompleted ? COLORS.textLight : COLORS.textPrimary,
                    letterSpacing: '0.2px',
                    textAlign: 'center',
                    lineHeight: '1.2'
                  }}>
                    {habit.label}
                  </span>
                  {isCompleted && !habit.isBoolean && duration && (
                    <span style={{
                      fontSize: '0.75em',
                      fontWeight: '600',
                      color: isCompleted ? COLORS.textLight : COLORS.textPrimary,
                      textAlign: 'center'
                    }}>
                      {formatMetricValue(duration, habit)}
                    </span>
                  )}
                </div>
              );
            })}
          </div>

          <div style={{
            marginTop: 'auto',
            display: 'flex',
            flexDirection: 'column',
            gap: '4px'
          }}>
            <ProgressBar
              value={completedCount}
              max={HABITS.length}
              color={getCompletionColor(completionPercentage)}
            />
            <div style={{
              textAlign: 'right',
              fontSize: '0.8em',
              fontWeight: '600',
              color: getCompletionColor(completionPercentage)
            }}>
              {completionPercentage}%
            </div>
          </div>
        </>
      )}
    </div>
  );
};

const MetricCard = ({ habit, current, previous, ytdTotal }) => {
  const trend = calculateTrendPercentage(current, previous);
  const formattedTotal = formatMetricValue(current, habit);
  const formattedYTD = formatMetricValue(ytdTotal, habit);

  return (
    <div style={{
      backgroundColor: 'var(--background-primary)',
      borderRadius: '16px',
      padding: '24px',
      boxShadow: 'var(--shadow-s)',
      display: 'flex',
      flexDirection: 'column',
      gap: '16px'
    }}>
      <div style={{
        display: 'flex',
        alignItems: 'center',
        gap: '12px'
      }}>
        <div style={{
          fontSize: '32px',
          backgroundColor: COLORS.secondary,
          borderRadius: '12px',
          padding: '12px',
          display: 'flex',
          alignItems: 'center',
          justifyContent: 'center'
        }}>
          {habit.emoji}
        </div>
        <div>
          <h3 style={{ margin: 0 }}>{habit.label}</h3>
          <div style={{
            color: 'var(--text-muted)',
            fontSize: '0.9em'
          }}>
            Last 30 Days
          </div>
        </div>
      </div>

      <div style={{
        display: 'grid',
        gridTemplateColumns: 'repeat(2, 1fr)',
        gap: '16px'
      }}>
        <div style={{
          backgroundColor: COLORS.secondary,
          padding: '16px',
          borderRadius: '12px',
          textAlign: 'center'
        }}>
          <div style={{ fontSize: '0.9em', color: 'var(--text-muted)' }}>Current</div>
          <div style={{
            fontSize: '1.4em',
            fontWeight: 'bold',
            marginTop: '4px'
          }}>
            {formattedTotal}
          </div>
        </div>

        <div style={{
          backgroundColor: COLORS.secondary,
          padding: '16px',
          borderRadius: '12px',
          textAlign: 'center'
        }}>
          <div style={{ fontSize: '0.9em', color: 'var(--text-muted)' }}>YTD</div>
          <div style={{
            fontSize: '1.4em',
            fontWeight: 'bold',
            marginTop: '4px'
          }}>
            {formattedYTD}
          </div>
        </div>
      </div>

      <div style={{
        display: 'flex',
        alignItems: 'center',
        justifyContent: 'space-between',
        backgroundColor: COLORS.secondary,
        padding: '12px 16px',
        borderRadius: '12px'
      }}>
        <span>Trend</span>
        <TrendIndicator current={current} previous={previous} />
      </div>
    </div>
  );
};

const TrendsView = ({ trends }) => {
  const monthlyProgress = (trends.currentMonth.perfectDays / GOALS.perfectDays.monthly) * 100;
  const yearlyProgress = (trends.yearToDate.perfectDays / GOALS.perfectDays.yearly) * 100;

  return (
    <div style={{
      padding: '24px',
      display: 'flex',
      flexDirection: 'column',
      gap: '32px'
    }}>
      <div style={{
        display: 'grid',
        gridTemplateColumns: 'repeat(auto-fit, minmax(280px, 1fr))',
        gap: '24px'
      }}>
        <div style={{
          backgroundColor: 'var(--background-primary)',
          borderRadius: '16px',
          padding: '24px',
          display: 'flex',
          alignItems: 'center',
          gap: '24px',
          boxShadow: 'var(--shadow-s)'
        }}>
          <CircularProgress value={monthlyProgress} size={100} color={COLORS.primary} />
          <div>
            <h3 style={{ margin: '0 0 8px 0', color: 'var(--text-normal)' }}>Monthly Goal</h3>
            <div style={{ fontSize: '1.2em', fontWeight: 'bold', color: 'var(--text-normal)' }}>
              {trends.currentMonth.perfectDays}/{GOALS.perfectDays.monthly} Perfect Days
            </div>
            <div style={{ color: 'var(--text-muted)' }}>
              {monthlyProgress.toFixed(1)}% Complete
            </div>
          </div>
        </div>

        <div style={{
          backgroundColor: 'var(--background-primary)',
          borderRadius: '16px',
          padding: '24px',
          display: 'flex',
          alignItems: 'center',
          gap: '24px',
          boxShadow: 'var(--shadow-s)'
        }}>
          <CircularProgress value={yearlyProgress} size={100} color={COLORS.progressBar.high} />
          <div>
            <h3 style={{ margin: '0 0 8px 0', color: 'var(--text-normal)' }}>Yearly Goal</h3>
            <div style={{ fontSize: '1.2em', fontWeight: 'bold', color: 'var(--text-normal)' }}>
              {trends.yearToDate.perfectDays}/{GOALS.perfectDays.yearly} Perfect Days
            </div>
            <div style={{ color: 'var(--text-muted)' }}>
              {yearlyProgress.toFixed(1)}% Complete
            </div>
          </div>
        </div>

        <div style={{
          backgroundColor: 'var(--background-primary)',
          borderRadius: '16px',
          padding: '24px',
          display: 'flex',
          alignItems: 'center',
          gap: '24px',
          boxShadow: 'var(--shadow-s)'
        }}>
          <div style={{
            width: '100px',
            height: '100px',
            display: 'flex',
            alignItems: 'center',
            justifyContent: 'center',
            fontSize: '48px',
            backgroundColor: 'var(--background-secondary)',
            borderRadius: '50%'
          }}>
            🔥
          </div>
          <div>
            <h3 style={{ margin: '0 0 8px 0', color: 'var(--text-normal)' }}>Current Streak</h3>
            <div style={{ fontSize: '1.2em', fontWeight: 'bold', color: 'var(--text-normal)' }}>
              {trends.last30Days.perfectDays} Days
            </div>
            <div style={{ color: 'var(--text-muted)' }}>
              Last 30 Days
            </div>
          </div>
        </div>
      </div>

      <div style={{
        display: 'grid',
        gridTemplateColumns: 'repeat(auto-fit, minmax(300px, 1fr))',
        gap: '24px'
      }}>
        {HABITS.map(habit => (
          <MetricCard
            key={habit.id}
            habit={habit}
            current={trends.last30Days.habitMetrics[habit.id].total}
            previous={trends.last30Days.habitMetrics[habit.id].previousPeriodTotal}
            ytdTotal={trends.yearToDate.habitMetrics[habit.id].total}
          />
        ))}
      </div>
    </div>
  );
};

const HistoricalView = ({
  sortedNotes,
  currentPage,
  setCurrentPage,
  updateHabit,
  getHabitStatus,
  getHabitDuration,
  editingTime,
  setEditingTime,
  updateHabitDuration,
  calculateCompletedHabits
}) => {
  const itemsPerPage = 20;
  const totalPages = Math.ceil(sortedNotes.length / itemsPerPage);
  const startIndex = currentPage * itemsPerPage;
  const displayNotes = sortedNotes.slice(startIndex, startIndex + itemsPerPage);

  return (
    <div style={{
      padding: '24px',
      backgroundColor: COLORS.secondary,
      borderRadius: '12px',
      marginTop: '24px'
    }}>
      <h3 style={{ margin: '0 0 20px 0' }}>Historical Data</h3>

      <div style={{
        width: '100%',
        overflow: 'auto',
        borderRadius: '12px',
        boxShadow: 'var(--shadow-s)',
        backgroundColor: 'var(--background-primary)'
      }}>
        <table style={{
          width: '100%',
          borderCollapse: 'separate',
          borderSpacing: 0
        }}>
          <thead>
            <tr>
              <th style={{
                padding: '16px',
                backgroundColor: COLORS.secondary,
                color: COLORS.textPrimary,
                fontWeight: 'bold',
                textAlign: 'left',
                position: 'sticky',
                top: 0,
                zIndex: 10,
              }}>Date</th>
              {HABITS.map(habit => (
                <th key={habit.id} style={{
                  padding: '16px',
                  backgroundColor: COLORS.secondary,
                  color: COLORS.textPrimary,
                  fontWeight: 'bold',
                  textAlign: 'center',
                  position: 'sticky',
                  top: 0,
                  zIndex: 10,
                }}>
                  <div style={{ fontSize: '1.4em' }}>{habit.emoji}</div>
                  <div>{habit.label}</div>
                </th>
              ))}
              <th style={{
                padding: '16px',
                backgroundColor: COLORS.secondary,
                color: COLORS.textPrimary,
                fontWeight: 'bold',
                textAlign: 'center',
                position: 'sticky',
                top: 0,
                zIndex: 10,
              }}>Completion</th>
            </tr>
          </thead>
          <tbody>
            {displayNotes.map((entry, index) => (
              <tr key={entry.$path} style={{
                backgroundColor: index % 2 === 0 ? 'var(--background-primary)' : 'var(--background-secondary)'
              }}>
                <td style={{
                  padding: '12px 16px',
                  borderBottom: '1px solid rgba(0, 0, 0, 0.05)',
                  minWidth: '150px'
                }}>{entry.$name}</td>
                {HABITS.map(habit => {
                  const isCompleted = getHabitStatus(entry, habit.id);
                  return (
                    <td key={habit.id} style={{
                      padding: '12px 16px',
                      borderBottom: '1px solid rgba(0, 0, 0, 0.05)',
                      textAlign: 'center',
                      minWidth: '120px'
                    }}>
                      <div style={{
                        display: 'flex',
                        flexDirection: 'column',
                        alignItems: 'center',
                        gap: '4px'
                      }}>
                        <div
                          onClick={() => updateHabit(entry, habit.id)}
                          style={{
                            padding: '4px 8px',
                            borderRadius: '4px',
                            backgroundColor: isCompleted ? COLORS.primary : COLORS.secondary,
                            color: isCompleted ? COLORS.textLight : 'var(--text-muted)',
                            cursor: 'pointer'
                          }}
                        >
                          {isCompleted ? '✓' : '×'}
                        </div>
                        {isCompleted && (
                          <TimeInput
                            entry={entry}
                            habitId={habit.id}
                            editingTime={editingTime}
                            setEditingTime={setEditingTime}
                            updateHabitDuration={updateHabitDuration}
                            getHabitStatus={getHabitStatus}
                            getHabitDuration={getHabitDuration}
                          />
                        )}
                      </div>
                    </td>
                  );
                })}
                <td style={{
                  padding: '12px 16px',
                  borderBottom: '1px solid rgba(0, 0, 0, 0.05)',
                  textAlign: 'center',
                  color: getCompletionColor(Math.round((calculateCompletedHabits(entry) / HABITS.length) * 100)),
                  fontWeight: '600'
                }}>
                  {Math.round((calculateCompletedHabits(entry) / HABITS.length) * 100)}%
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>

      <div style={{
        display: 'flex',
        justifyContent: 'center',
        gap: '8px',
        marginTop: '16px'
      }}>
        <ActionButton
          icon="←"
          onClick={() => setCurrentPage(prev => Math.max(0, prev - 1))}
          extraStyles={{
            opacity: currentPage === 0 ? 0.5 : 1,
            cursor: currentPage === 0 ? 'default' : 'pointer'
          }}
        />
        <span style={{
          padding: '8px 16px',
          backgroundColor: 'var(--background-primary)',
          borderRadius: '8px'
        }}>
          Page {currentPage + 1} of {totalPages}
        </span>
        <ActionButton
          icon="→"
          onClick={() => setCurrentPage(prev => Math.min(totalPages - 1, prev + 1))}
          extraStyles={{
            opacity: currentPage === totalPages - 1 ? 0.5 : 1,
            cursor: currentPage === totalPages - 1 ? 'default' : 'pointer'
          }}
        />
      </div>
    </div>
  );
};

const GoalsView = ({ entries, daysInMonth }) => {
  const habitsWithGoals = HABITS.filter(h => h.monthlyGoal);

  const calculateProgress = (habitId) => {
    const total = entries.reduce((sum, entry) => sum + (entry?.value(habitId) ?? 0), 0);
    const habit = HABITS.find(h => h.id === habitId);

    // Count days that actually have data
    const daysWithData = entries.filter(entry => {
      const value = entry?.value(habitId);
      return value !== null && value !== undefined && value > 0;
    }).length;

    // Calculate progress against monthly goal
    const progress = (total / habit.monthlyGoal) * 100;

    // Calculate daily average based on days with actual data, or 1 if no days have data
    const daysForAverage = Math.max(daysWithData, 1);
    const dailyAverage = Number((total / daysForAverage).toFixed(2));

    // Project monthly total based on daily average
    const projection = Number((dailyAverage * daysInMonth).toFixed(2));
    const isOnTrack = projection >= habit.monthlyGoal;

    return {
      total,
      progress: Number(Math.min(progress, 100).toFixed(2)),
      dailyAverage,
      projection,
      isOnTrack,
      daysWithData
    };
  };

  const getProgressGradient = (isOnTrack) => {
    return {
      start: isOnTrack ? COLORS.progressBar.gradient.start : COLORS.progressBar.gradient.start,
      end: isOnTrack ? COLORS.progressBar.gradient.end : COLORS.progressBar.gradient.end
    };
  };

  return (
    <div style={{
      padding: '16px',
      width: '100%',
      maxWidth: '100%'
    }}>
      <div style={{
        display: 'grid',
        gridTemplateColumns: 'repeat(auto-fit, minmax(450px, 1fr))',
        gap: '24px',
        width: '100%'
      }}>
        {habitsWithGoals.map(habit => {
          const stats = calculateProgress(habit.id);
          const progressColor = stats.isOnTrack ? COLORS.progressBar.high : COLORS.progressBar.low;
          const gradient = getProgressGradient(stats.isOnTrack);

          return (
            <div key={habit.id} style={{
              background: 'var(--background-secondary)',
              borderRadius: '12px',
              padding: '24px',
              boxShadow: 'var(--shadow-s)',
              minWidth: '450px',
              flex: '1 1 auto'
            }}>
              <div style={{
                display: 'flex',
                alignItems: 'center',
                gap: '16px',
                marginBottom: '16px'
              }}>
                <div style={{
                  width: '100px',
                  height: '100px',
                  position: 'relative'
                }}>
                  <CircularProgress
                    progress={stats.progress}
                    size={100}
                    strokeWidth={10}
                    circleColor={progressColor}
                    gradientStart={gradient.start}
                    gradientEnd={gradient.end}
                  />
                  <div style={{
                    position: 'absolute',
                    top: '50%',
                    left: '50%',
                    transform: 'translate(-50%, -50%)',
                    fontSize: '42px',
                    display: 'flex',
                    alignItems: 'center',
                    justifyContent: 'center',
                    width: '80%',
                    height: '80%'
                  }}>
                    {habit.emoji}
                  </div>
                </div>

                <div style={{ flex: 1 }}>
                  <h3 style={{
                    margin: '0 0 8px 0',
                    textAlign: 'center',
                    fontSize: '1.4em',
                    color: 'var(--text-normal)'
                  }}>{habit.label}</h3>
                  <div style={{
                    display: 'grid',
                    gridTemplateColumns: '1fr 1fr',
                    gap: '12px'
                  }}>
                    <Stat
                      label="Current"
                      value={formatMetricValue(stats.total, habit)}
                    />
                    <Stat
                      label="Goal"
                      value={formatMetricValue(habit.monthlyGoal, habit)}
                    />
                    <Stat
                      label="Daily Avg"
                      value={formatMetricValue(stats.dailyAverage, habit)}
                    />
                    <Stat
                      label="Projected"
                      value={formatMetricValue(stats.projection, habit)}
                      color={stats.isOnTrack ? 'var(--color-green)' : 'var(--color-red)'}
                    />
                  </div>
                </div>
              </div>

              <div style={{
                height: '40px',
                background: 'var(--background-primary)',
                borderRadius: '20px',
                overflow: 'hidden',
                position: 'relative'
              }}>
                <div style={{
                  position: 'absolute',
                  top: '0',
                  left: '0',
                  height: '100%',
                  width: `${stats.progress}%`,
                  background: `linear-gradient(90deg, ${gradient.start} 0%, ${gradient.end} 100%)`,
                  transition: 'width 0.3s ease'
                }} />
                <div style={{
                  position: 'absolute',
                  top: '50%',
                  left: '50%',
                  transform: 'translate(-50%, -50%)',
                  color: 'var(--text-normal)',
                  fontWeight: 'bold'
                }}>
                  {stats.progress}%
                </div>
              </div>
            </div>
          );
        })}
      </div>
    </div>
  );
};

const WeeklyGoalsView = ({ entries }) => {
  const habitsWithGoals = HABITS.filter(h => h.monthlyGoal);
  const daysInWeek = 7;

  const calculateWeeklyProgress = (habitId) => {
    const total = entries.reduce((sum, entry) => sum + (entry?.value(habitId) ?? 0), 0);
    const habit = HABITS.find(h => h.id === habitId);

    // Count days that actually have data
    const daysWithData = entries.filter(entry => {
      const value = entry?.value(habitId);
      return value !== null && value !== undefined && value > 0;
    }).length;

    // Calculate weekly goal as a proportion of monthly goal
    const weeklyGoal = Math.round(habit.monthlyGoal * (daysInWeek / 30));
    const progress = (total / weeklyGoal) * 100;

    // Calculate daily average based on days with actual data, or 1 if no days have data
    const daysForAverage = Math.max(daysWithData, 1);
    const dailyAverage = Number((total / daysForAverage).toFixed(2));

    const projection = Number((dailyAverage * daysInWeek).toFixed(2));
    const isOnTrack = projection >= weeklyGoal;

    return {
      total,
      weeklyGoal,
      progress: Number(Math.min(progress, 100).toFixed(2)),
      dailyAverage,
      projection,
      isOnTrack,
      daysWithData
    };
  };

  const getProgressGradient = (isOnTrack) => {
    return {
      start: isOnTrack ? COLORS.progressBar.gradient.start : COLORS.progressBar.gradient.start,
      end: isOnTrack ? COLORS.progressBar.gradient.end : COLORS.progressBar.gradient.end
    };
  };

  return (
    <div style={{
      padding: '16px',
      width: '100%',
      maxWidth: '100%'
    }}>
      <div style={{
        display: 'grid',
        gridTemplateColumns: 'repeat(auto-fit, minmax(450px, 1fr))',
        gap: '24px',
        width: '100%'
      }}>
        {habitsWithGoals.map(habit => {
          const stats = calculateWeeklyProgress(habit.id);
          const progressColor = stats.isOnTrack ? COLORS.progressBar.high : COLORS.progressBar.low;
          const gradient = getProgressGradient(stats.isOnTrack);

          return (
            <div key={habit.id} style={{
              background: 'var(--background-secondary)',
              borderRadius: '12px',
              padding: '24px',
              boxShadow: 'var(--shadow-s)',
              minWidth: '450px',
              flex: '1 1 auto'
            }}>
              <div style={{
                display: 'flex',
                alignItems: 'center',
                gap: '16px',
                marginBottom: '16px'
              }}>
                <div style={{
                  width: '100px',
                  height: '100px',
                  position: 'relative'
                }}>
                  <CircularProgress
                    progress={stats.progress}
                    size={100}
                    strokeWidth={10}
                    circleColor={progressColor}
                    gradientStart={gradient.start}
                    gradientEnd={gradient.end}
                  />
                  <div style={{
                    position: 'absolute',
                    top: '50%',
                    left: '50%',
                    transform: 'translate(-50%, -50%)',
                    fontSize: '42px',
                    display: 'flex',
                    alignItems: 'center',
                    justifyContent: 'center',
                    width: '80%',
                    height: '80%'
                  }}>
                    {habit.emoji}
                  </div>
                </div>

                <div style={{ flex: 1 }}>
                  <h3 style={{
                    margin: '0 0 8px 0',
                    textAlign: 'center',
                    fontSize: '1.4em',
                    color: 'var(--text-normal)'
                  }}>{habit.label}</h3>
                  <div style={{
                    display: 'grid',
                    gridTemplateColumns: '1fr 1fr',
                    gap: '12px'
                  }}>
                    <Stat
                      label="Current"
                      value={formatMetricValue(stats.total, habit)}
                    />
                    <Stat
                      label="Weekly Goal"
                      value={formatMetricValue(stats.weeklyGoal, habit)}
                    />
                    <Stat
                      label="Daily Avg"
                      value={formatMetricValue(stats.dailyAverage, habit)}
                    />
                    <Stat
                      label="Projected"
                      value={formatMetricValue(stats.projection, habit)}
                      color={stats.isOnTrack ? 'var(--color-green)' : 'var(--color-red)'}
                    />
                  </div>
                </div>
              </div>

              <div style={{
                height: '40px',
                background: 'var(--background-primary)',
                borderRadius: '20px',
                overflow: 'hidden',
                position: 'relative'
              }}>
                <div style={{
                  position: 'absolute',
                  top: '0',
                  left: '0',
                  height: '100%',
                  width: `${stats.progress}%`,
                  background: `linear-gradient(90deg, ${gradient.start} 0%, ${gradient.end} 100%)`,
                  transition: 'width 0.3s ease'
                }} />
                <div style={{
                  position: 'absolute',
                  top: '50%',
                  left: '50%',
                  transform: 'translate(-50%, -50%)',
                  color: 'var(--text-normal)',
                  fontWeight: 'bold'
                }}>
                  {stats.progress}%
                </div>
              </div>
            </div>
          );
        })}
      </div>
    </div>
  );
};

const Stat = ({ label, value, color }) => (
  <div style={{ textAlign: 'center' }}>
    <div style={{
      fontSize: '0.9em',
      color: 'var(--text-muted)',
      marginBottom: '4px'
    }}>
      {label}
    </div>
    <div style={{
      fontSize: '1.1em',
      fontWeight: 'bold',
      color: color || 'var(--text-normal)'
    }}>
      {value}
    </div>
  </div>
);

function HabitTracker() {
  // State Management
  const [activeView, setActiveView] = dc.useState('weekly');
  const [selectedDate, setSelectedDate] = dc.useState(dc.luxon.DateTime.now());
  const [editingTime, setEditingTime] = dc.useState(null);
  const [currentPage, setCurrentPage] = dc.useState(0);

  // Data Queries and Utility Functions
  const dailyNotes = dc.useQuery(`
    @page 
    AND path("${JOURNAL_PATH}")
  `);

  const sortedNotes = dc.useMemo(() => {
    return [...dailyNotes].sort((a, b) => b.$name.localeCompare(a.$name));
  }, [dailyNotes]);

  const getNotesForPeriod = (startDate) => {
    return sortedNotes.filter(note => {
      const noteDate = dc.luxon.DateTime.fromISO(note.$name);
      return noteDate >= startDate;
    });
  };

  // Add new function to get notes for specific date range
  const getNotesForDateRange = (startDate, endDate) => {
    return sortedNotes.filter(note => {
      const noteDate = dc.luxon.DateTime.fromISO(note.$name);
      // Use startOf('day') and endOf('day') to ensure full day coverage
      return noteDate >= startDate.startOf('day') && noteDate <= endDate.endOf('day');
    });
  };

  const last30DaysNotes = dc.useMemo(() =>
    getNotesForPeriod(selectedDate.minus({ days: 30 })),
    [sortedNotes, selectedDate]
  );

  const yearToDateNotes = dc.useMemo(() =>
    getNotesForPeriod(selectedDate.startOf('year')),
    [sortedNotes, selectedDate]
  );

  const currentMonthNotes = dc.useMemo(() =>
    getNotesForPeriod(selectedDate.startOf('month')),
    [sortedNotes, selectedDate]
  );

  const previousMonthNotes = dc.useMemo(() =>
    sortedNotes.filter(note => {
      const noteDate = dc.luxon.DateTime.fromISO(note.$name);
      const monthAgo = selectedDate.minus({ months: 1 });
      return noteDate >= monthAgo && noteDate < selectedDate.startOf('month');
    }),
    [sortedNotes, selectedDate]
  );

  const getHabitStatus = (entry, habitId) => {
    const habit = HABITS.find(h => h.id === habitId);
    const value = entry?.value(habitId) ?? 0;
    return value >= habit.defaultDuration;
  };

  const getHabitDuration = (entry, habitId) => {
    return entry?.value(habitId) ?? null;
  };

  const calculateCompletedHabits = (entry) => {
    if (!entry) return 0;
    return HABITS.reduce((count, habit) =>
      count + (getHabitStatus(entry, habit.id) ? 1 : 0), 0);
  };

  const calculatePerfectDays = (notes) => {
    return notes.reduce((count, note) =>
      count + (calculateCompletedHabits(note) === HABITS.length ? 1 : 0), 0);
  };

  const calculateTrends = () => {
    const trends = {
      last30Days: {
        perfectDays: calculatePerfectDays(last30DaysNotes),
        habitMetrics: {}
      },
      yearToDate: {
        perfectDays: calculatePerfectDays(yearToDateNotes),
        habitMetrics: {}
      },
      currentMonth: {
        perfectDays: calculatePerfectDays(currentMonthNotes),
        progress: 0
      }
    };

    trends.currentMonth.progress = (trends.currentMonth.perfectDays / GOALS.perfectDays.monthly) * 100;

    HABITS.forEach(habit => {
      const last30Total = last30DaysNotes.reduce((sum, note) => {
        const value = note?.value(habit.id);
        const numValue = value ? Number(value) : 0;
        return sum + (isNaN(numValue) ? 0 : numValue);
      }, 0);

      const ytdTotal = yearToDateNotes.reduce((sum, note) => {
        const value = note?.value(habit.id);
        const numValue = value ? Number(value) : 0;
        return sum + (isNaN(numValue) ? 0 : numValue);
      }, 0);

      const previousMonthTotal = previousMonthNotes.reduce((sum, note) => {
        const value = note?.value(habit.id);
        const numValue = value ? Number(value) : 0;
        return sum + (isNaN(numValue) ? 0 : numValue);
      }, 0);

      trends.last30Days.habitMetrics[habit.id] = {
        total: last30Total,
        previousPeriodTotal: previousMonthTotal
      };

      trends.yearToDate.habitMetrics[habit.id] = {
        total: ytdTotal
      };
    });

    return trends;
  };

  // Get current week's notes based on selected date
  const currentWeekNotes = dc.useMemo(() =>
    sortedNotes.filter(note => {
      const noteDate = dc.luxon.DateTime.fromISO(note.$name);
      const startOfWeek = selectedDate.startOf('week');
      const endOfWeek = selectedDate.endOf('week');
      return noteDate >= startOfWeek && noteDate <= endOfWeek;
    }),
    [sortedNotes, selectedDate]
  );

  // Action Handlers
  async function updateHabit(entry, habitId) {
    const file = app.vault.getAbstractFileByPath(entry.$path);
    await app.fileManager.processFrontMatter(file, (frontmatter) => {
      const habit = HABITS.find(h => h.id === habitId);
      const currentValue = frontmatter[habitId];
      frontmatter[habitId] = currentValue ? 0 : habit.defaultDuration;
    });
  }

  async function updateHabitDuration(entry, habitId, duration) {
    const file = app.vault.getAbstractFileByPath(entry.$path);
    await app.fileManager.processFrontMatter(file, (frontmatter) => {
      frontmatter[habitId] = parseInt(duration) || 0;
    });
    setEditingTime(null);
  }

  const navigateDate = (direction) => {
    setSelectedDate(prev => prev.plus({ days: direction }));
  };

  // Main Layout
  return (
  <div style={{
    width: '100%',
    margin: '0 auto',
    padding: '24px',
    display: 'flex',
    flexDirection: 'column',
    gap: '24px'
  }}>
    <NavigationControls
      selectedDate={selectedDate}
      navigateDate={navigateDate}
      activeView={activeView}
      setActiveView={setActiveView}
    />

    <StyledCard>
      <CalendarView
        selectedDate={selectedDate}
        sortedNotes={getNotesForDateRange(selectedDate.minus({ days: 6 }), selectedDate)}
        getHabitStatus={getHabitStatus}
        calculateCompletedHabits={calculateCompletedHabits}
        updateHabit={updateHabit}
        getHabitDuration={getHabitDuration}
        editingTime={editingTime}
        setEditingTime={setEditingTime}
        updateHabitDuration={updateHabitDuration}
      />

      <div style={{
        display: 'flex',
        justifyContent: 'center',
        gap: '16px',
        marginTop: '16px',
        paddingTop: '16px',
        borderTop: '1px solid var(--background-modifier-border)'
      }}>
        <ActionButton
          icon="📅"
          onClick={() => setActiveView(activeView === 'weekly' ? null : 'weekly')}
          isActive={activeView === 'weekly'}
          extraStyles={{ padding: '12px' }}
        />
        <ActionButton
          icon="🚀"
          onClick={() => setActiveView(activeView === 'goals' ? null : 'goals')}
          isActive={activeView === 'goals'}
          extraStyles={{ padding: '12px' }}
        />
        <ActionButton
          icon="🚧"
          onClick={() => setActiveView(activeView === 'stats' ? null : 'stats')}
          isActive={activeView === 'stats'}
          extraStyles={{ padding: '12px' }}
        />
        <ActionButton
          icon="🎯"
          onClick={() => setActiveView(activeView === 'history' ? null : 'history')}
          isActive={activeView === 'history'}
          extraStyles={{ padding: '12px' }}
        />
      </div>
    </StyledCard>

    {activeView === 'weekly' && (
      <WeeklyGoalsView entries={currentWeekNotes} />
    )}
    {activeView === 'goals' && (
      <GoalsView
        entries={currentMonthNotes}
        daysInMonth={new Date(new Date().getFullYear(), new Date().getMonth() + 1, 0).getDate()}
      />
    )}
    {activeView === 'stats' && <TrendsView trends={calculateTrends()} />}
    {activeView === 'history' && (
      <HistoricalView
        sortedNotes={sortedNotes}
        currentPage={currentPage}
        setCurrentPage={setCurrentPage}
        updateHabit={updateHabit}
        getHabitStatus={getHabitStatus}
        getHabitDuration={getHabitDuration}
        editingTime={editingTime}
        setEditingTime={setEditingTime}
        updateHabitDuration={updateHabitDuration}
        calculateCompletedHabits={calculateCompletedHabits}
      />
    )}
  </div>
);
}

return HabitTracker;
```