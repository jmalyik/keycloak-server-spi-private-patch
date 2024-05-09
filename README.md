# Keycloak Server Spi Private Patch

## Keycloak version

This patch has been made for version 16.1.1

## The bugs that requires this Patch

### My problem

We are using ConditionalOTP (one time password) in browser flow.

To use it, we need to define a keycloak role and associate one or more LDAP group with it.

We are using LDAP_ONLY mode in group-ldap-mapper in ldap federation configuration.

## Where can be a solution too

https://keycloak.discourse.group/t/performance-issue-with-large-number-of-ldap-group/7740

# Explanation

For some reason, keycloak tries to check the group membership in the evaluation of the ConditionalOTP condition by walking the group tree and check the tree path that in case of lots of groups can be really slow.

The code that does this breadth-first search is in org.keycloak.models.utils.KeycloakModelUtils::findGroupByPath function in org.keycloak:keycloak-server-spi-private:16.1.1 artifact.

So, the idea behind this path is the following: instead of doing this BFS we have to collect all groups with the name of the last group of the group path then calculate and check their paths for matching.


The code that has to be replaced:

<pre>
      return realm.getTopLevelGroupsStream().map(group -> { // instead of search for exact match we iterate over
            String groupName = group.getName();
            String[] pathSegments = formatPathSegments(split, 0, groupName);

            if (groupName.equals(pathSegments[0])) {
                if (pathSegments.length == 1) {
                    return group;
                }
                else {
                    if (pathSegments.length > 1) {
                        GroupModel subGroup = findSubGroup(pathSegments, 1, group); // findSubGroup - call it recursively
                        if (subGroup != null) return subGroup;
                    }
                }

            }
            return null;
        }).filter(Objects::nonNull).findFirst().orElse(null);
</pre>

The new version:

<pre>
        return = realm.searchForGroupByNameStream(lastGroupNameInPath, null, null) // find for exact match 
        		.filter(group -> checkGroupPath(group, searchedPath)) // check only the matching groups
        		.findFirst().orElse(null);
</pre>

The new version requires some new functions:

<pre>
    private static boolean checkGroupPath(GroupModel group, String searchedPath) {
    	String groupPath = getGroupPath(group);
    	return searchedPath.equals(groupPath);
	}

	private static String getGroupPath(GroupModel group) {
    	if(group.getParent() != null) {
    		return getGroupPath(group.getParent()) + "/" + group.getName();
    	} else {
    		return group.getName();
    	}
	}
</pre>

## Installation

Just replace the ${KEYCLOAK_HOME}/modules/system/layers/keycloak/org/keycloak/keycloak-server-spi-private/main/keycloak-server-spi-private-16.1.1.jar jar within the keycloak docker image with the target/keycloak-server-spi-private-patch-16.1.1.jar file - I used it in this way.