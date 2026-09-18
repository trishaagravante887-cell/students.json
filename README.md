const fs = require('fs');
const path = require('path');

// ==========================================
// 1. Helper & Core Functions
// ==========================================

/**
 * Calculates the average grade for a single student.
 * Handles edge cases like missing grades array or empty grades.
 */
function getAverageGrade(student) {
  if (!student || !Array.isArray(student.grades) || student.grades.length === 0) {
    return 0;
  }
  const sum = student.grades.reduce((acc, g) => acc + g, 0);
  return Number((sum / student.grades.length).toFixed(2));
}

/**
 * Returns the top n students sorted by average grade in descending order.
 */
function getTopStudents(students, n) {
  if (!Array.isArray(students)) throw new TypeError('First argument must be an array of students.');
  if (typeof n !== 'number' || n < 0) throw new Error('Parameter "n" must be a non-negative number.');

  return [...students]
    .sort((a, b) => getAverageGrade(b) - getAverageGrade(a))
    .slice(0, n);
}

/**
 * Groups all students by their course field.
 */
function groupByCourse(students) {
  if (!Array.isArray(students)) throw new TypeError('Argument must be an array of students.');

  return students.reduce((acc, student) => {
    const course = student.course || 'Unassigned';
    if (!acc[course]) {
      acc[course] = [];
    }
    acc[course].push(student);
    return acc;
  }, {});
}

/**
 * Returns a count of how many students are currently enrolled vs not enrolled.
 */
function getEnrolledCount(students) {
  if (!Array.isArray(students)) throw new TypeError('Argument must be an array of students.');

  return students.reduce(
    (acc, student) => {
      if (student.enrolled) {
        acc.enrolled += 1;
      } else {
        acc.notEnrolled += 1;
      }
      return acc;
    },
    { enrolled: 0, notEnrolled: 0 }
  );
}

/**
 * Performs a case-insensitive search for a student by name.
 */
function findStudent(students, name) {
  if (!Array.isArray(students)) throw new TypeError('First argument must be an array of students.');
  if (typeof name !== 'string') throw new TypeError('Search name must be a string.');

  const target = name.trim().toLowerCase();
  const match = students.find((s) => s.name && s.name.toLowerCase() === target);
  return match || null;
}

/**
 * Returns the average grade for each course, sorted from highest to lowest.
 */
function getCourseAverages(students) {
  if (!Array.isArray(students)) throw new TypeError('Argument must be an array of students.');

  const grouped = groupByCourse(students);

  const courseAvgs = Object.keys(grouped).map((course) => {
    const courseStudents = grouped[course];
    const totalAvg = courseStudents.reduce((sum, s) => sum + getAverageGrade(s), 0);
    const courseAvg = courseStudents.length > 0 ? Number((totalAvg / courseStudents.length).toFixed(2)) : 0;

    return { course, averageGrade: courseAvg };
  });

  return courseAvgs.sort((a, b) => b.averageGrade - a.averageGrade);
}

/**
 * Builds a single summary object containing overall metrics and course breakdown.
 */
function exportSummary(students) {
  if (!Array.isArray(students)) throw new TypeError('Argument must be an array of students.');

  if (students.length === 0) {
    return {
      totalStudents: 0,
      overallAverage: 0,
      topStudent: null,
      courseBreakdown: []
    };
  }

  const totalAvgSum = students.reduce((sum, s) => sum + getAverageGrade(s), 0);
  const overallAverage = Number((totalAvgSum / students.length).toFixed(2));
  const topStudent = getTopStudents(students, 1)[0] || null;

  return {
    totalStudents: students.length,
    overallAverage,
    topStudent: topStudent
      ? { id: topStudent.id, name: topStudent.name, averageGrade: getAverageGrade(topStudent) }
      : null,
    courseBreakdown: getCourseAverages(students)
  };
}

// ==========================================
// 2. Main Execution
// ==========================================

function main() {
  const jsonPath = path.join(__dirname, 'students.json');
  const reportPath = path.join(__dirname, 'report.json');

  let students = [];

  // Load dataset from disk
  try {
    const rawData = fs.readFileSync(jsonPath, 'utf8');
    students = JSON.parse(rawData);
  } catch (err) {
    console.error(`Error loading ${jsonPath}:`, err.message);
    process.exit(1);
  }

  console.log('=============== STUDENT RECORDS REPORT ===============\n');

  // 1. Total Count & Enrolled Status
  const enrolledCounts = getEnrolledCount(students);
  console.log(`Total Students: ${students.length}`);
  console.log(`Enrolled: ${enrolledCounts.enrolled} | Not Enrolled: ${enrolledCounts.notEnrolled}\n`);

  // 2. Overall Course Averages
  console.log('--- Course Averages (Highest to Lowest) ---');
  const courseAverages = getCourseAverages(students);
  courseAverages.forEach((c) => console.log(`  * ${c.course}: ${c.averageGrade}`));
  console.log('');

  // 3. Top 3 Students
  console.log('--- Top 3 Students ---');
  const topStudents = getTopStudents(students, 3);
  topStudents.forEach((s, i) => {
    console.log(`  ${i + 1}. ${s.name} (${s.course}) - Avg: ${getAverageGrade(s)}`);
  });
  console.log('');

  // 4. Sample Search
  console.log('--- Sample Search ---');
  const searchName = 'Alice Johnson';
  const found = findStudent(students, searchName);
  console.log(`Searching for "${searchName}":`, found ? `Found ID ${found.id}` : 'Not Found');
  console.log('');

  // 5. Generate and Write Summary Report
  const summary = exportSummary(students);
  fs.writeFileSync(reportPath, JSON.stringify(summary, null, 2), 'utf8');
  console.log(`Summary report exported to ${reportPath}`);
  console.log('\n=====================================================');
}

main();
