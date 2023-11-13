// Dependencies
// import com.atlassian.greenhopper.api.rank.RankService
import com.atlassian.jira.component.ComponentAccessor
import com.atlassian.jira.issue.MutableIssue
import com.atlassian.jira.issue.Issue
import com.atlassian.jira.issue.IssueManager
import com.atlassian.jira.issue.IssueInputParameters
import com.atlassian.jira.issue.IssueFactory
import com.atlassian.jira.issue.util.DefaultIssueChangeHolder
import com.atlassian.jira.issue.util.IssueChangeHolder
import com.atlassian.jira.issue.link.IssueLink
import com.atlassian.jira.issue.link.IssueLinkManager
import com.atlassian.jira.issue.link.IssueLinkType
import com.atlassian.jira.issue.link.IssueLinkTypeManager
import com.atlassian.jira.issue.issuetype.IssueType
import com.atlassian.jira.config.IssueTypeManager
import com.atlassian.jira.project.Project
import com.atlassian.jira.project.ProjectManager
import com.atlassian.jira.security.JiraAuthenticationContext
import com.atlassian.jira.user.ApplicationUser

// Defining custom variables necessary for the execution of the script
String projectKey = "PROJECT";
String prefix = "";
String fieldName = "Checklist";

/**
 * Possible values:
 * {Epic}
 * {Task}
 * {Sub-task}
 */
String issueTypeToCreate = "Epic";

/**
 * Possible values:
 * {Blocks}
 * {Cloners}
 * {Duplicate}
 * {Epic-Story Link}
 * {jira_subtask_link}
 * {Parent-Child Link}
 * {Problem/Incident}
 * {Relates}
 */
String issueLinkTypeToCreate = "Parent-Child Link";

/**
 * Create an empty issue
 * @param {ApplicationUser} user - The user that will create the issue, the user must have the right permission to do it
 * @param {Project} project - The project where you want to create the issue
 * @param {IssueType} issueType - The issue type you want to create
 */
def createMutableIssue(user, project, issueType) {
  log.info("Create Mutable Issue");
  // Get the IssueFactory, it's a tool used to create an empty issue object
  def issueFactory = ComponentAccessor.getComponentOfType(IssueFactory.class);

  // Create the empty issue
  def newIssue = issueFactory.getIssue();

  log.info("  Project Id: ${project.id}");
  // Assign a project id to the issue
  newIssue.projectId = project.id;

  log.info("  IssueType Id: ${issueType.getId()}");
  // Assign an issue type to the issue
  newIssue.issueTypeId = issueType.getId();

  // Assign a temporary value to the summary, if something went wrong we could debug the script from here
  newIssue.summary = "[TECHNICAL VALUE] Mutable Issue";

  // After the creation of the issue is done, we return the issue object
  return newIssue;
}

/**
 * Create the issue and set all properties and link it to the parent (summary, type)
 * @param {ApplicationUser} user - The user that will create the issue, the user must have the right permission to do it
 * @param {Project} project - The project where you want to create the issue
 * @param {IssueType} issueType - The issue type you want to create
 * @param {String} summary - The summary of the issue
 */
def createIssue(user, project, issueType, summary) {
  MutableIssue mutableIssue = createMutableIssue(user, project, issueType);

  mutableIssue.setSummary(summary);

  return mutableIssue;
}

/**
 * Persit the issue into Jira
 * @param {MutableIssue} issue - The new issue to save
 * @param {ApplicationUser} user - The user that will create the issue, the user must have the right permission to do it
 */
def saveNewIssue(issue, user) {
  log.info("Save Mutable Issue");
  ComponentAccessor.issueManager.createIssueObject(
    user,
    issue
  )
  
  return issue;
}

/**
 * Link the new issue with its parent
 * @param {MutableIssue} parentIssue - The current issue from where the script is triggered
 * @param {MutableIssue} subIssue - The created/new issue
 * @param {IssueLinkType} issueLinkType - The type of type you need to use
 * @param {ApplicationUser} user - The user that will create the issue, the user must have the right permission to do it
 */
def linkSubIssue(parentIssue, subIssue, issueLinkType, user) {
  log.info("");
  log.info("Link New Issue with Parent");
  log.info("  ParentIssue Id ${parentIssue.getId()}");
  log.info("  Issue Id ${subIssue.getId()}");
  log.info("  IssueLinkType Id ${issueLinkType.getId()}");
  ComponentAccessor.issueLinkManager.createIssueLink(parentIssue.getId(), subIssue.getId(), issueLinkType.getId(), 1L, user);
}

/**
 * Parent-Child Link type have a specific way of working, if you need to see the new issue in Child issues in the parent, you need to apply this method
 * @param {MutableIssue} currentIssue - The current issue from where the script is triggered
 * @param {MutableIssue} newIssue - The created/new issue
 */
def setParentChildLink(currentIssue, newIssue) {
  def parentLinkField = ComponentAccessor.customFieldManager.getCustomFieldObjectByName("Parent Link");
  def parentLinkFieldType = parentLinkField.getCustomFieldType();
  def newParentLink = parentLinkFieldType.getSingularObjectFromString(currentIssue.key);
  newIssue.setCustomFieldValue(parentLinkField, newParentLink);
}

/**
 * Avoid duplicating issues by summary (regular links)
 * @param {MutableIssue} currentIssue - The current issue from where the script is triggered
 * @param {String} summary - The summary of the new issue
 */
def isAlreadyCreatedRegular(currentIssue) {
  Boolean isAlreadyCreated = false;

  List<Issue> linkedIssues = currentIssue.getLinkedIssues();
  
  for (Issue linkedIssue : linkedIssues) {
    if (!isAlreadyCreated && linkedIssue && linkedIssue.getSummary() == summary) {
      isAlreadyCreated = true;
    }
  }

  return isAlreadyCreated;
}

/**
 * Avoid duplicating issues by summary (Parent-Child Link)
 * @param {MutableIssue} currentIssue - The current issue from where the script is triggered
 * @param {String} issueLinkTypeToCreate - THe issue link type you need to create
 * @param {String} summary - The summary of the new issue
 */
def isAlreadyCreatedParentChild(currentIssue, issueLinkTypeToCreate, summary) {
  Boolean isAlreadyCreated = false;

  def linkType = ComponentAccessor.getOSGiComponentInstanceOfType(IssueLinkTypeManager.class).getIssueLinkTypesByName(issueLinkTypeToCreate).first();
  
  // Get all the issue links of the specified type for the parent issue
  def issueLinks = ComponentAccessor.getIssueLinkManager().getOutwardLinks(currentIssue.id);

  // Filter the links to get the child issues
  def childIssues = issueLinks.findAll { it.issueLinkType == linkType }

  // Iterate through child issues
  childIssues.each { childLink ->
    def childIssue = childLink.destinationObject

    if (!isAlreadyCreated && childIssue && childIssue.getSummary() == summary) {
      isAlreadyCreated = true;
    }
  }

  return isAlreadyCreated;
}

// Get the current user, this user must have permissions to create issues
ApplicationUser loggedInUser = ComponentAccessor.jiraAuthenticationContext.loggedInUser;

// Just reassign the global issue variable to a more comprehensive variable
MutableIssue currentIssue = issue;

// Get managers
// IssueLinkManager issueLinkManager = ComponentAccessor.getIssueLinkManager();
ProjectManager projectManager = ComponentAccessor.getProjectManager();

// Get all issueTypes of the instance, necessary to get its ID
Collection<IssueType> issueTypes = ComponentAccessor.getOSGiComponentInstanceOfType(IssueTypeManager.class).getIssueTypes();
// Get the specific issueType from the list by its name
IssueType issueType = issueTypes.find { it.getName() == issueTypeToCreate }

// Get all issueTypeLinks of the instance, necessary to get its ID
Collection<IssueLinkType> issueTypeLinks = ComponentAccessor.getOSGiComponentInstanceOfType(IssueLinkTypeManager.class).getIssueLinkTypes(false);
// Get the specific issueLinkType from the list by its name
IssueLinkType issueLinkType = issueTypeLinks.find { it.getName() == issueLinkTypeToCreate }

// Get the project where we want to create the subtask
Project project = projectManager.getProjectObjByKey(projectKey);

// Get the list of each values checked from the field
List<String> items = issue.get(fieldName);

/**
 * Loop on each values and create a issue
 */
log.info("Creating Issues from $fieldName field ${items}");
// MutableIssue previousIssue = null;
for (String item : items) {
  // Construct the summary
  def summary = "${prefix}${item}";
  log.info("  Summary: ${summary}");

  // CHeck is the new issue already exists
  Boolean isAlreadyCreated = false;
  // If we have to handle a Parent-Child Link we need to check the children of the parent differently
  if (issueLinkTypeToCreate == "Parent-Child Link") {
    isAlreadyCreated = isAlreadyCreatedParentChild(currentIssue, issueLinkTypeToCreate, summary);
  } else {
    isAlreadyCreated = isAlreadyCreatedRegular(currentIssue, summary);
  }

  // If we found an issue with the same summary, we skip issue creation
  if (isAlreadyCreated) {
    log.info("Aborting creation of Issue: $summary");
  } else {
    log.info("");
    log.info("Creating Issue: $summary");
    MutableIssue newIssue = createIssue(currentUser, project, issueType, summary);

    // If we have to handle a Parent-Child Link we need to apply this
    if (issueLinkTypeToCreate == "Parent-Child Link") {
      setParentChildLink(currentIssue, newIssue);
    }

    saveNewIssue(newIssue, currentUser);

    linkSubIssue(currentIssue, newIssue, issueLinkType, currentUser);
    
    // def rankField = ComponentAccessor.customFieldManager.getCustomFieldObjectByName("Rank");
    // log.info(rankField);
    //if (previousIssue != null) {
    //  rankService.rankBefore(currentUser, rankField.id, previousIssue, createdIssue);
    // }
    
    // previousIssue = createdIssue;
  }
}


