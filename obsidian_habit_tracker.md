```datacorejsx
// Constants and color configuration
const HABITS = [
  { id: 'reading', emoji: '📚', label: 'Pages', defaultDuration: 25, unit: 'pages' },
  { id: 'meditation', emoji: '🧘‍♂️', label: 'Meditation', defaultDuration: 20, unit: 'minutes' },
  { id: 'weightlifting', emoji: '🏋️', label: 'Weight Lifting', defaultDuration: 45, unit: 'minutes' },
  { id: 'run', emoji: '🏃‍♂️', label: 'Run', defaultDuration: 2, unit: 'miles' },
  { id: 'writing', emoji: '✍️', label: 'Writing', defaultDuration: 30, unit: 'minutes' },
  { id: 'sleep', emoji: '😴', label: 'Sleep', defaultDuration: 8, unit: 'hours' }
];

const GOALS = {
  perfectDays: {
    monthly: 20,
    yearly: 250
  }
};

const DAYS = ['Sun', 'Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat'];

const COLORS = {
  primary: '#5B8AF5',
  secondary: '#F5F7FA',
  hoverState: '#4A7AE5',
  textPrimary: '#2C3E50',
  textLight: '#FFFFFF',
  progressBar: {
    low: '#FF6B6B',
    medium: '#FFD93D',
    high: '#4CD964'
  }
};

// Utility Functions
const formatMetricValue = (value, habit) => {
  switch(habit.unit) {
    case 'minutes':
      return value === 1 ? '1 Minute' : `${value} Minutes`;
    case 'miles':
      return value === 1 ? '1 Mile' : `${value} Miles`;
    case 'pages':
      return value === 1 ? '1 Page' : `${value} Pages`;
    case 'hours':
      return value === 1 ? '1 Hour' : `${value} Hours`;
    default:
      return value;
  }
};

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
  const radius = (size - 8) / 2;
  const circumference = 2 * Math.PI * radius;
  const progress = ((100 - value) / 100) * circumference;

  return (
    <svg width={size} height={size} style={{ transform: 'rotate(-90deg)' }}>
      <circle
        cx={size / 2}
        cy={size / 2}
        r={radius}
        stroke="var(--background-modifier-border)"
        strokeWidth="4"
        fill="none"
      />
      <circle
        cx={size / 2}
        cy={size / 2}
        r={radius}
        stroke={color}
        strokeWidth="4"
        strokeDasharray={circumference}
        strokeDashoffset={progress}
        fill="none"
        style={{ transition: 'stroke-dashoffset 0.5s ease' }}
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
    color = '#4caf50';
    indicator = '↑';
  } else if (trend < 0) {
    color = '#f44336';
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
      backgroundColor: 'rgba(0, 0, 0, 0.05)',
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
      boxShadow: '0 2px 8px rgba(0, 0, 0, 0.1)'
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
        background: 'white',
        padding: '8px 16px',
        borderRadius: '8px',
        boxShadow: 'inset 0 2px 4px rgba(0, 0, 0, 0.05)'
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
  
  // Show 6 days (today + previous 5 days)
  for (let i = 0; i < 6; i++) {
    dates.push(currentDate.minus({ days: i }));
  }

  const rows = [];
  for (let i = 0; i < dates.length; i += 3) {
    rows.push(dates.slice(i, i + 3));
  }

  const notesMap = new Map(sortedNotes.map(note => [note.$name, note]));
  const today = dc.luxon.DateTime.now().startOf('day');

  return (
    <div style={{ display: 'flex', flexDirection: 'column', gap: '16px' }}>
      {rows.map((row, rowIndex) => (
        <div
          key={rowIndex}
          style={{
            display: 'grid',
            gridTemplateColumns: 'repeat(3, 1fr)',
            gap: '16px'
          }}
        >
          {row.map((date) => {
            const dateStr = date.toFormat('yyyy-MM-dd');
            const entry = notesMap.get(dateStr);
            const isSelected = date.hasSame(selectedDate, 'day');
            const isToday = date.hasSame(today, 'day');

            return (
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
            );
          })}
        </div>
      ))}
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

  return (
    <div style={{
      padding: '12px',
      borderRadius: '16px',
      backgroundColor: 'white',
      color: COLORS.textPrimary,
      boxShadow: '0 2px 12px rgba(0, 0, 0, 0.06)',
      border: isSelected ? `2px solid ${COLORS.primary}` : '1px solid rgba(0, 0, 0, 0.05)',
      transition: 'all 0.3s cubic-bezier(0.4, 0, 0.2, 1)',
      display: 'flex',
      flexDirection: 'column',
      gap: '8px',
      minHeight: '168px'
    }}>
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

      {entry && (
        <>
          <div style={{
            display: 'grid',
            gridTemplateColumns: 'repeat(2, 1fr)',
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
                    padding: '8px 12px',
                    width: '100%',
                    height: '100%',
                    minHeight: '40px',
                    transition: 'all 0.2s ease'
                  }}
                >
                  <span style={{ 
                    fontSize: '20px',
                    marginBottom: '2px'
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
                  {isCompleted && duration && (
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
      backgroundColor: 'white',
      borderRadius: '16px',
      padding: '24px',
      boxShadow: '0 2px 8px rgba(0, 0, 0, 0.1)',
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
          backgroundColor: 'white',
          borderRadius: '16px',
          padding: '24px',
          display: 'flex',
          alignItems: 'center',
          gap: '24px'
        }}>
          <CircularProgress value={monthlyProgress} size={100} color={COLORS.primary} />
          <div>
            <h3 style={{ margin: '0 0 8px 0' }}>Monthly Goal</h3>
            <div style={{ fontSize: '1.2em', fontWeight: 'bold' }}>
              {trends.currentMonth.perfectDays}/{GOALS.perfectDays.monthly} Perfect Days
            </div>
            <div style={{ color: 'var(--text-muted)' }}>
              {monthlyProgress.toFixed(1)}% Complete
            </div>
          </div>
        </div>

        <div style={{
          backgroundColor: 'white',
          borderRadius: '16px',
          padding: '24px',
          display: 'flex',
          alignItems: 'center',
          gap: '24px'
        }}>
          <CircularProgress value={yearlyProgress} size={100} color={COLORS.progressBar.high} />
          <div>
            <h3 style={{ margin: '0 0 8px 0' }}>Yearly Goal</h3>
            <div style={{ fontSize: '1.2em', fontWeight: 'bold' }}>
              {trends.yearToDate.perfectDays}/{GOALS.perfectDays.yearly} Perfect Days
            </div>
            <div style={{ color: 'var(--text-muted)' }}>
              {yearlyProgress.toFixed(1)}% Complete
            </div>
          </div>
        </div>

        <div style={{
          backgroundColor: 'white',
          borderRadius: '16px',
          padding: '24px',
          display: 'flex',
          alignItems: 'center',
          gap: '24px'
        }}>
          <div style={{
            width: '100px',
            height: '100px',
            display: 'flex',
            alignItems: 'center',
            justifyContent: 'center',
            fontSize: '48px',
            backgroundColor: COLORS.secondary,
            borderRadius: '50%'
          }}>
            🔥
          </div>
          <div>
            <h3 style={{ margin: '0 0 8px 0' }}>Current Streak</h3>
            <div style={{ fontSize: '1.2em', fontWeight: 'bold' }}>
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
        boxShadow: '0 2px 8px rgba(0, 0, 0, 0.1)',
        backgroundColor: 'white'
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
                backgroundColor: index % 2 === 0 ? 'white' : COLORS.secondary
              }}>
                <td style={{
                  padding: '12px 16px',
                  borderBottom: '1px solid rgba(0, 0, 0, 0.05)'
                }}>{entry.$name}</td>
                {HABITS.map(habit => {
                  const isCompleted = getHabitStatus(entry, habit.id);
                  return (
                    <td key={habit.id} style={{
                      padding: '12px 16px',
                      borderBottom: '1px solid rgba(0, 0, 0, 0.05)',
                      textAlign: 'center'
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
          backgroundColor: 'white',
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

function HabitTracker() {
  // State Management
  const [selectedDate, setSelectedDate] = dc.useState(dc.luxon.DateTime.now());
  const [activeView, setActiveView] = dc.useState(null);
  const [editingTime, setEditingTime] = dc.useState(null);
  const [currentPage, setCurrentPage] = dc.useState(0);

  // Data Queries and Utility Functions
  const dailyNotes = dc.useQuery(`
    @page 
    AND path("Notes/Daily Notes")
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
    const habits = entry?.value('habits');
    return habits?.[habitId] ?? false;
  };

  const getHabitDuration = (entry, habitId) => {
    const habits = entry?.value('habits');
    return habits?.[`${habitId}_duration`] ?? null;
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
      const last30Total = last30DaysNotes.reduce((sum, note) => 
        sum + (getHabitDuration(note, habit.id) || 0), 0);
      
      const ytdTotal = yearToDateNotes.reduce((sum, note) => 
        sum + (getHabitDuration(note, habit.id) || 0), 0);

      const previousMonthTotal = previousMonthNotes.reduce((sum, note) => 
        sum + (getHabitDuration(note, habit.id) || 0), 0);

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

  // Action Handlers
  async function updateHabit(entry, habitId) {
    const file = app.vault.getAbstractFileByPath(entry.$path);
    await app.fileManager.processFrontMatter(file, (frontmatter) => {
      if (!frontmatter.habits) frontmatter.habits = {};
      const newStatus = !frontmatter.habits[habitId];
      frontmatter.habits[habitId] = newStatus;
      
      if (newStatus) {
        const habit = HABITS.find(h => h.id === habitId);
        frontmatter.habits[`${habitId}_duration`] = habit.defaultDuration;
      }
    });
  }

  async function updateHabitDuration(entry, habitId, duration) {
    const file = app.vault.getAbstractFileByPath(entry.$path);
    await app.fileManager.processFrontMatter(file, (frontmatter) => {
      if (!frontmatter.habits) frontmatter.habits = {};
      frontmatter.habits[`${habitId}_duration`] = parseInt(duration) || 0;
    });
    setEditingTime(null);
  }

  const navigateDate = (direction) => {
    setSelectedDate(prev => prev.plus({ days: direction }));
  };

  // Main Layout
  return (
  <div style={{ 
    maxWidth: '1200px', 
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
        sortedNotes={sortedNotes.slice(0, 6)}
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
          icon="📊"
          onClick={() => setActiveView(activeView === 'stats' ? null : 'stats')}
          isActive={activeView === 'stats'}
          extraStyles={{ padding: '12px' }}
        />
        <ActionButton
          icon="📚"
          onClick={() => setActiveView(activeView === 'history' ? null : 'history')}
          isActive={activeView === 'history'}
          extraStyles={{ padding: '12px' }}
        />
      </div>
    </StyledCard>

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
